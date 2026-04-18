# How LLM Models Run on Mobile Hardware

> **Based strictly on the source code of:**
> - [`google-ai-edge/LiteRT-LM`](https://github.com/google-ai-edge/LiteRT-LM)
> - [`google-ai-edge/gallery`](https://github.com/google-ai-edge/gallery)

---

## Overview

LiteRT-LM is Google's on-device LLM runtime. The Gallery Android app uses it to run models like Gemma directly on the phone. Below is a code-by-code walkthrough of every resource-handling decision: model loading, RAM allocation, KV cache, delegate/backend selection, threading, and cleanup.

---

## 1. Model File Loading — How the File Gets Into RAM

**File:** `runtime/util/executor_settings_base.h` → `ModelAssets`
```cpp
class ModelAssets {
  static absl::StatusOr<ModelAssets> Create(absl::string_view model_path);
  static absl::StatusOr<ModelAssets> Create(std::shared_ptr<ScopedFile> model_file);
  static absl::StatusOr<ModelAssets> Create(std::shared_ptr<MemoryMappedFile> model_file);
  static absl::StatusOr<ModelAssets> Create(std::shared_ptr<DataStream> data_stream);
};
```

There are **four ways** to open the model:

| Method | RAM Impact |
|---|---|
| `model_path` string | File opened lazily; OS memory-maps it on access |
| `ScopedFile` (file descriptor) | Kernel manages the file; no extra copy |
| `MemoryMappedFile` | `mmap()` — model weights land in virtual address space only, not physical RAM until accessed |
| `DataStream` | Used for streaming (e.g. over network on desktop) |

**File:** `runtime/util/memory_mapped_file.h`
```cpp
class MemoryMappedFile {
  static absl::StatusOr<std::unique_ptr<MemoryMappedFile>> Create(
      absl::string_view path);     // read-only mmap
  static absl::StatusOr<std::unique_ptr<MemoryMappedFile>> CreateMutable(
      absl::string_view path);     // writable mmap
  virtual uint64_t length() = 0;
  virtual void* data() = 0;       // pointer directly into the mmap region
};
```

**Key point:** The model weights are NOT fully read into a C++ `vector`. They are memory-mapped. The OS loads only the pages that are actually touched during inference. This is critical on mobile where physical RAM is scarce.

**File:** `runtime/core/engine_impl.cc` — `EngineImpl::Create()`
```cpp
ASSIGN_OR_RETURN(auto model_resources,
                 BuildLiteRtCompiledModelResources(model_assets));
```
This parses the `.litertlm` container and extracts embedded TFLite subgraphs (prefill graph, decode graph), tokenizer, and metadata.

---

## 2. Hardware Backend Selection

**File:** `runtime/executor/executor_settings_base.h`
```cpp
enum class Backend {
  UNSPECIFIED,
  CPU_ARTISAN,          // Hand-written CPU path
  GPU_ARTISAN,          // Hand-written GPU path
  CPU,                  // LiteRT (TFLite + XNNPACK)
  GPU,                  // LiteRT (OpenCL / Metal / WebGPU)
  GOOGLE_TENSOR_ARTISAN,// Google Tensor NPU (Pixel)
  NPU,                  // Generic NPU via LiteRT dispatch
};
```

Selection starts from the app layer:

**File:** `runtime/engine/engine_settings.h`
```cpp
static absl::StatusOr<EngineSettings> CreateDefault(
    ModelAssets model_assets,
    Backend backend = Backend::CPU,          // default: CPU
    std::optional<Backend> vision_backend = std::nullopt,
    std::optional<Backend> audio_backend    = std::nullopt,
    std::optional<Backend> sampler_backend  = std::nullopt);
```

The engine then validates the backend against the embedded model metadata:
```cpp
RETURN_IF_ERROR(engine_settings.MaybeUpdateAndValidate(
    tokenizer.get(), llm_metadata, input_prompt_as_hint,
    model_resources->GetTFLiteModelBackendConstraint(ModelType::kTfLitePrefillDecode),
    ...));
```
`GetTFLiteModelBackendConstraint()` reads a constraint string baked into the model file (e.g. `"GPU_ARTISAN_ONLY"`) to override the caller's choice if needed.

**For NPU specifically**, the runtime sets up a dispatch library directory:
```cpp
// engine_impl.cc — NPU path
if (!main_executor_settings.GetLitertDispatchLibDir().empty()) {
    env_options.push_back(::litert::Environment::Option{
        ::litert::Environment::OptionTag::DispatchLibraryDir,
        main_executor_settings.GetLitertDispatchLibDir()});
}
```
This is the `.so` that bridges LiteRT to the chip vendor's NPU SDK.

**Singleton LiteRT Environment** (one per process lifetime):
```cpp
// engine_impl.cc
static absl::NoDestructor<absl::StatusOr<Environment>> kEnvironment(
    [&]() -> absl::StatusOr<Environment> {
        ...
        LITERT_ASSIGN_OR_RETURN(auto env, Environment::Create(env_options));
        return std::move(env);
    }());
```
GPU context is never re-created between calls. This saves the expensive GPU driver initialization on every inference.

---

## 3. RAM Allocation — What Lives Where in Memory

### 3a. Model Weights
Memory-mapped (see §1). The OS page-cache holds the most recently used weight pages. Android's OOM killer can reclaim these pages when RAM is needed — model weights do **not** have to be fully resident.

### 3b. KV Cache — The Biggest RAM Consumer

**File:** `runtime/executor/kv_cache_interface.h`
```cpp
class KVCacheInterface {
  virtual int GetNumEntries() const = 0;   // tokens stored
  virtual int GetBatchSize() const = 0;    // parallel sequences
  virtual absl::StatusOr<std::string> Serialize() const = 0;  // save to disk
  virtual absl::Status Load(absl::string_view serialized_kv_cache) = 0;
  virtual absl::StatusOr<std::unique_ptr<KVCacheInterface>> DeepCopy() const = 0;
};
```

The **maximum KV cache size** (= maximum context length) is set in:

**File:** `runtime/executor/llm_executor_settings.h`
```cpp
class LlmExecutorSettings {
  uint32_t max_num_tokens_;  // total (input + output) tokens stored in KV cache
  void SetMaxNumTokens(uint32_t max_num_tokens) { ... }
};
```
If `max_num_tokens = 512` and the model has 18 transformer layers, each with K and V tensors of size `[1, 512, num_heads, head_dim]`, that is the full KV RAM footprint — allocated once during engine creation.

### 3c. CPU — Dynamic KV Cache Growth

**File:** `runtime/executor/llm_executor_settings.h`
```cpp
struct CpuConfig {
  uint32_t kv_increment_size = 16;  // grow KV cache by 16 tokens per decode step
  int prefill_chunk_size = -1;      // -1 = no chunking; chunking reduces peak RAM
  uint32_t number_of_threads = 4;   // CPU inference threads
};
```
On CPU the KV cache grows dynamically in 16-token increments. This avoids pre-allocating the full context up front.

### 3d. GPU — Upfront Allocation

**File:** `runtime/executor/llm_executor_settings.h`
```cpp
struct GpuConfig {
  uint32_t max_top_k = 1;        // greedy only by default
  bool external_tensor_mode = false;
};

struct GpuArtisanConfig {
  uint32_t num_decode_steps_per_sync = 1;
  uint32_t sequence_batch_size = 0;  // 0 = programmatically optimized
  bool wait_for_weight_uploads = false;
  bool convert_weights_on_gpu = true;   // (via AdvancedSettings)
};
```
For GPU the entire KV cache is pre-allocated in GPU VRAM when the executor is created. `max_num_tokens` therefore directly determines GPU memory usage.

### 3e. Weight Cache (Disk-Backed, Reduces Init Time)

**File:** `runtime/executor/executor_settings_base.h`
```cpp
// CPU (XNNPACK):
static constexpr absl::string_view kXnnpackCacheSuffix = ".xnnpack_cache";
// GPU (MlDrift):
static constexpr absl::string_view kMlDriftCacheSuffix = "_mldrift_program_cache.bin";
```
First run: XNNPACK re-packs weights (into a cache file), GPU compiles shaders (into program cache). Second run: loads from cache, startup is much faster. These files live in the app's private cache dir.

### 3f. Input/Output Tensors

These are `litert::TensorBuffer` objects allocated by the compiled model. Each prefill/decode step writes input token IDs into an input tensor and reads output logits from an output tensor. The sizes are fixed by the model signature (e.g. `[1, 512]` for CPU or `[1, 1]` for single-step decode).

---

## 4. Interpreter / Delegate Setup (LlmLiteRtCompiledModelExecutor)

**File:** `runtime/executor/llm_litert_compiled_model_executor.h`
```cpp
class LlmLiteRtCompiledModelExecutorBase : public LlmExecutor {
  absl::Status Prefill(const ExecutorInputs& inputs) override;
  absl::StatusOr<std::vector<std::vector<int>>> Decode() override;
  absl::Status Decode(const ExecutorInputs& inputs, TensorBuffer& output_logits) override;
  absl::StatusOr<TensorBuffer> DecodeLogits(const ExecutorInputs& inputs) override;
};
```

The executor wraps a `litert::CompiledModel` which handles the underlying TFLite interpreter + delegate. Delegate assignment happens inside LiteRT based on the `Backend` enum and the `Environment` options set at engine creation.

**File:** `runtime/core/engine_impl.cc` — delegate/executor dispatch:
```cpp
switch (main_executor_settings.GetBackend()) {
  default: {
    ASSIGN_OR_RETURN(executor,
                     CreateLlmLiteRtCompiledModelExecutor(
                         main_executor_settings, env, *model_resources));
  }
}
```
A single `switch` dispatches to the right executor factory. LiteRT internally partitions ops: ops supported by the delegate (GPU/NPU) run on that accelerator; fallback ops run on CPU. This is **delegate clustering**, which can be disabled:
```cpp
struct AdvancedSettings {
  bool disable_delegate_clustering = false; // default: clustering ON
};
```

---

## 5. Threading

### Engine Thread Pool (Inference Serialization)

**File:** `runtime/framework/threadpool.cc`
```cpp
ThreadPool::ThreadPool(const std::string& name_prefix, size_t max_num_threads, ...)
    : max_num_threads_(max_num_threads == 0 ? 1 : max_num_threads) { ... }
```

**File:** `runtime/core/engine_impl.cc`
```cpp
auto worker_thread_pool =
    std::make_unique<ThreadPool>(/*name_prefix=*/"engine",
                                 /*max_num_threads=*/1);
```
The engine uses **exactly one background thread** named `"engine"`. All prefill and decode tasks are serialized through this single worker thread. This prevents concurrent model calls from corrupting the KV cache.

The ThreadPool uses pthreads on Android/Linux:

**File:** `runtime/framework/worker_thread_pthread.cc` — spins up a `pthread` per worker, blocked on a `mutex_.Await()` condition variable until a task is queued.

### CPU Inference Threads (XNNPACK)

```cpp
struct CpuConfig {
  uint32_t number_of_threads = 4; // default 4 XNNPACK compute threads
};
```
XNNPACK internally spawns 4 pthreads (or as configured) to parallelize matrix multiplications across CPU cores. These are separate from the "engine" serialization thread.

### GPU Upload Threads

```cpp
struct AdvancedSettings {
  int num_threads_to_upload = -1;   // -1 = runtime decides
  int num_threads_to_compile = -1;  // -1 = runtime decides
};
```

---

## 6. Gallery Android App — How the Kotlin Layer Uses All of This

**File:** `Android/src/app/src/main/java/com/google/ai/edge/gallery/runtime/LlmModelHelper.kt`

```kotlin
interface LlmModelHelper {
  // Step 1: Create the LiteRT-LM Engine + first Session
  fun initialize(
    context: Context,
    model: Model,
    supportImage: Boolean,
    supportAudio: Boolean,
    onDone: (String) -> Unit,
    systemInstruction: Contents? = null,
    tools: List<ToolProvider> = listOf(),
    enableConversationConstrainedDecoding: Boolean = false,
    coroutineScope: CoroutineScope? = null,
  )

  // Step 2: Send a message and stream back the response
  fun runInference(
    model: Model,
    input: String,
    resultListener: ResultListener,  // (partialResult, done, thinkingResult) -> Unit
    cleanUpListener: CleanUpListener,
    onError: (message: String) -> Unit = {},
    images: List<Bitmap> = listOf(),
    audioClips: List<ByteArray> = listOf(),
    coroutineScope: CoroutineScope? = null,
    extraContext: Map<String, String>? = null,
  )

  // Step 3: Stop mid-generation
  fun stopResponse(model: Model)

  // Step 4: Start fresh conversation (resets KV cache via new Session)
  fun resetConversation(model: Model, ...)

  // Step 5: Free all resources when done
  fun cleanUp(model: Model, onDone: () -> Unit)
}
```

The call flow maps directly to the C++ engine:

```
Kotlin: LlmModelHelper.initialize()
  └─ C++: EngineImpl::Create()
       ├─ BuildLiteRtCompiledModelResources()   → mmap the .litertlm file
       ├─ GetEnvironment()                       → create LiteRT env (singleton, GPU context)
       ├─ create_tokenizer() [async if new fmt]  → load SentencePiece/BPE tokenizer
       ├─ CreateLlmLiteRtCompiledModelExecutor() → load model, apply delegate, alloc KV cache
       └─ ThreadPool("engine", 1)               → single serializing worker thread

Kotlin: LlmModelHelper.runInference()
  └─ C++: Session::GenerateContentStream()
       ├─ Session::RunPrefill()   → tokenize input, fill KV cache
       └─ Session::RunDecode()    → auto-regressive decode loop
            └─ callback(partialResult) → streamed to Kotlin resultListener

Kotlin: LlmModelHelper.cleanUp()
  └─ C++: ~EngineImpl()
       └─ worker_thread_pool_->WaitUntilDone(kDefaultTimeout=10min)
            └─ ~ThreadPool() → joins all pthreads
```

---

## 7. Resource Cleanup and Lifecycle

**File:** `runtime/core/engine_impl.cc`
```cpp
class EngineImpl : public Engine {
 public:
  ~EngineImpl() override {
    auto status = WaitUntilDone(Engine::kDefaultTimeout);  // wait up to 10 min
    if (!status.ok()) {
      ABSL_LOG(ERROR) << "Failed to wait for engine to finish: " << status;
    }
  }
};
```

**File:** `runtime/framework/threadpool.cc`
```cpp
ThreadPool::~ThreadPool() {
  {
    absl::MutexLock lock(mutex_);
    stopped_ = true;            // signal all workers to exit
    threads_to_join.swap(threads_);
  }
  for (auto& thread_ptr : threads_to_join) {
    thread_ptr->Join();         // pthread_join — block until thread exits
  }
}
```

Cleanup chain:
1. Kotlin calls `cleanUp(model, onDone)` → destroys the `Engine` smart pointer
2. `~EngineImpl()` waits for the worker thread pool to drain
3. `~ThreadPool()` signals all worker pthreads and joins them
4. `~LlmLiteRtCompiledModelExecutor()` releases GPU/NPU buffers and KV cache
5. `~ModelResources()` closes the mmap'd file handles
6. The OS unmaps the model weights from virtual address space

---

## 8. Memory Budget Summary (Concrete Example)

For Gemma 3n E2B-it (the 2B-parameter Gemma 3n model from `litert-community/gemma-3n-E2B-it-litert-lm`) with `max_num_tokens = 512`, CPU backend:

| Resource | Approximate Size | When Allocated |
|---|---|---|
| Model weights (mmap) | ~1.5 GB (4-bit quantized) | Virtual only; OS pages loaded on demand |
| KV cache (CPU) | ~50–200 MB (grows 16 tokens at a time) | During decode |
| Input/output tensors | ~2–4 MB | Fixed at executor creation |
| Tokenizer (SentencePiece) | ~1–4 MB | Engine init |
| XNNPACK weight cache | ~300–800 MB | Written to disk on first run, loaded later |
| ThreadPool overhead | < 1 MB | Engine init |

---

## 9. Key Design Decisions (Derived from Code)

1. **Single inference thread** — prevents race conditions on the KV cache. All token generation is sequential even if the app calls `GenerateContentStream()` concurrently.
2. **Memory-mapped weights** — model weights are never fully copied to heap. This is the main technique that makes multi-GB models viable on 4–8 GB mobile devices.
3. **Incremental KV cache** (CPU) — the KV cache grows 16 tokens at a time rather than pre-allocating the full `max_num_tokens` × layers × heads × dims. Peak RAM is proportional to actual context used, not max context.
4. **GPU weight upload threads** — `num_threads_to_upload` uploads model weights to the GPU in background threads while the app is doing other work. `wait_for_weight_uploads = false` means inference can start before all weights are uploaded (pipelined).
5. **Singleton LiteRT Environment** — the GPU driver context is created once and reused for the process lifetime. Re-creating it would cost hundreds of milliseconds per inference call.
6. **Async tokenizer loading** — for new `.litertlm` model files that contain `LlmModelType`, the tokenizer is loaded in a separate `std::async` thread in parallel with executor creation to reduce cold-start latency.
