# Architecture & Resource Handling Analysis

## Does this repository contain any on-device ML model / inference code?

**No.** There is zero on-device machine-learning, model-inference, or AI runtime code anywhere
in this repository. Specifically, there is no:

- TensorFlow Lite / LiteRT interpreter or `.tflite` model file
- CoreML `.mlpackage` / `.mlmodel`
- ONNX Runtime
- PyTorch Mobile / ExecuTorch
- WebNN / WebGPU compute pipeline for model inference
- Any GPU/NPU delegate, tensor-allocation buffer, or KV-cache

The repository is a **React 18 single-page web application** (a Blinkit grocery-delivery clone)
that runs entirely inside a browser. All data is static JavaScript objects or stored in the
browser's `localStorage`. There is no server, no backend API, and no ML component.

---

## Actual Resource Handling Architecture

The sections below trace exactly how RAM and storage resources are acquired, held, and released,
with references to the specific files and functions that drive each path.

---

### 1. JavaScript Heap (RAM) — React State

**Entry point:** `src/index.js` → mounts `<App />` into the DOM.

`src/App.js` wraps the entire component tree in four context providers:

```
AuthProvider          (src/context/AuthContext.js)
  └─ CartProvider     (src/context/CartContext.js)
       └─ WishlistProvider  (src/context/WishlistContext.js)
            └─ LocationProvider  (src/context/LocationContext.js)
```

Each provider allocates a JavaScript object in the V8/JavaScriptCore heap via `useState`:

| Context file | `useState` variable | What it holds in RAM |
|---|---|---|
| `AuthContext.js` | `user`, `currentUser`, `loading` | One plain JS object for the logged-in user (< 1 KB) |
| `CartContext.js` | `cart` | Array of product objects, one entry per cart item |
| `WishlistContext.js` | `wishlist` | Array of product objects in the wishlist |
| `LocationContext.js` | *(location string)* | A single geolocation/pincode string |

These objects live in the browser tab's JavaScript heap for the entire page lifetime. React
re-renders write new object references; the garbage collector frees old ones automatically.

**Key functions that mutate RAM state:**

- `CartContext.js → addToCart(product, quantity)` — spreads a new array into state
  (`[...prevCart, { ...product, quantity }]`), causing the old array to become GC-eligible.
- `CartContext.js → removeFromCart(productId)` — filters the array, releasing the removed item.
- `CartContext.js → clearCart()` — replaces the cart with `[]`, releasing all item objects.
- `WishlistContext.js → addToWishlist(product)` / `removeFromWishlist(productId)` — same pattern.
- `AuthContext.js → logout()` — sets both `user` and `currentUser` to `null`, releasing the
  user object from the React tree.

**No manual memory management is needed** — the browser's garbage collector reclaims all
unreferenced objects automatically.

---

### 2. Persistent Storage — `localStorage`

`localStorage` is the only persistence layer. It is stored on the device's flash/SSD, not in
RAM. The per-origin quota is **browser-dependent**: Safari enforces ~5 MB, Chrome and Firefox
typically allow up to 10 MB, and other browsers vary. All reads/writes go through
the service classes in `src/services/`.

#### Data stored and where it is written

| `localStorage` key | Written by | Read by |
|---|---|---|
| `products` | `ProductService.initializeData()` (first run only) | `ProductService.getAllProducts()` |
| `categories` | `ProductService.initializeData()` (first run only) | `ProductService.getAllCategories()` |
| `cart` | `CartContext.js` `useEffect` (every cart change) | `CartContext.js` `useEffect` (on mount) |
| `wishlist` | `WishlistContext.js` `useEffect` (every wishlist change) | `WishlistContext.js` `useEffect` (on mount) |
| `orders` | `OrderService.createOrder()` / `updateOrderStatus()` | `OrderService.getAllOrders()` |
| `user` | `AuthContext.js → login()` / `adminLogin()` | `AuthContext.js` `useEffect` (on mount) |

#### How data enters RAM from storage

Both `CartContext.js` and `WishlistContext.js` use the identical pattern:

```js
// src/context/CartContext.js  (lines 16-21)
useEffect(() => {
  const storedCart = localStorage.getItem('cart');   // disk → string
  if (storedCart) {
    setCart(JSON.parse(storedCart));                  // string → JS array → React state (heap)
  }
}, []);
```

`JSON.parse` deserialises the stored string into a fresh JavaScript object graph in the heap.
The string itself is then GC-eligible.

#### How RAM state is persisted back to storage

```js
// src/context/CartContext.js  (lines 23-25)
useEffect(() => {
  localStorage.setItem('cart', JSON.stringify(cart)); // JS array → string → disk
}, [cart]);
```

This runs on every state change. `JSON.stringify` allocates a temporary string, writes it to
storage, and then the string becomes GC-eligible.

#### Cleanup path

`AuthContext.js → logout()` removes the `user` key:

```js
localStorage.removeItem('user');  // frees disk space immediately
```

`CartContext.js → clearCart()` sets `cart` to `[]`, which the `useEffect` above then
serialises as `"[]"` back to `localStorage`.

---

### 3. Product / Category Data — Static JS Module

All product catalogue data (100+ products, 10+ categories) is declared as plain JavaScript
arrays in `src/utils/constants.js`. When the JavaScript bundle is executed:

1. **Bundle load** — the browser parses the JS file; V8 allocates the object graph in the
   young-generation heap.
2. **`ProductService` constructor** (`src/services/productService.js → initializeData()`) —
   checks `localStorage`. If empty (first run), serialises the array to `localStorage` via
   `localStorage.setItem('products', JSON.stringify(PRODUCTS))`. The original in-module
   reference (`PRODUCTS`) remains alive for the page lifetime because it is a module-level
   constant.
3. All subsequent reads go through `localStorage`, so the module constant is only a fallback.

---

### 4. Order Service — `src/services/orderService.js`

`OrderService` follows the same `localStorage` pattern:

- **`initializeData()`** — seeds `orders` key with `[]` on first load.
- **`createOrder(orderData)`** — reads the full orders array (`JSON.parse`), pushes a new
  order object, then writes back (`JSON.stringify` + `setItem`). The entire array is briefly
  in RAM during this operation, then the temporary copy is GC-eligible.
- **`updateOrderStatus(orderId, newStatus)`** — reads, mutates in place, writes back.
- Neither function retains a long-lived reference to the data array; it is read, modified,
  serialised, and then becomes GC-eligible.

---

### 5. Performance Monitoring — `src/reportWebVitals.js`

The `reportWebVitals` function dynamically imports the `web-vitals` library and reports
Core Web Vitals metrics (CLS, FID, FCP, LCP, TTFB) to whatever callback is passed in. This
is a lightweight observer that does not allocate significant RAM or use any native device APIs
beyond the standard browser Performance Observer interface.

---

### 6. Browser-Side Hardware Resources

Since this is a browser application there are **no direct hardware API calls**. The browser
engine handles all of the following transparently:

| Resource | Who manages it | How |
|---|---|---|
| **CPU** | Browser JS engine (V8 / JSC) | Single-threaded JS event loop; React renders are synchronous; no Web Workers |
| **GPU** | Browser compositor | CSS transitions / animations in `App.css` use the GPU compositor layer automatically (e.g. `transform`, `opacity`); no WebGL or WebGPU |
| **Network** | Browser fetch stack | External images from Unsplash / CDN are loaded via `<img>` tags; the browser manages the HTTP connection pool and image decode memory |
| **Storage** | Browser `localStorage` | Quota is browser-dependent (~5 MB in Safari, up to 10 MB in Chrome/Firefox); no IndexedDB, OPFS, or native file I/O |
| **Threads** | None (app-level) | No Web Workers, SharedArrayBuffer, or Atomics are used |

---

## Summary

| Question | Answer |
|---|---|
| On-device ML model? | **None** |
| Tensor / weight buffer allocations? | **None** |
| GPU/NPU delegate? | **None** |
| KV-cache or inference threads? | **None** |
| RAM is managed by | Browser GC via React `useState` |
| Persistence is managed by | `localStorage` (≤ 10 MB, on-device flash) |
| Primary RAM consumers | React context state trees (`cart`, `wishlist`, `user`) and deserialized product arrays |
| Cleanup paths | `clearCart()`, `removeFromWishlist()`, `logout()`, and automatic GC on state replacement |
