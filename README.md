# Blinkit Clone - React Application

A full-featured e-commerce application built with React, Bootstrap, and local storage for data persistence.

## Features

- 🛒 Complete shopping cart functionality
- 💝 Wishlist management
- 👤 User authentication (simulated)
- 📍 Location-based delivery
- 🔍 Product search and filtering
- 📱 Responsive design
- 💳 Multi-step checkout process
- 📦 Order tracking
- 👨‍💼 Admin panel for product/order management
- 🏠 Address management
- 📊 Order analytics

## Tech Stack

- **Frontend**: React 18, React Router DOM
- **UI Framework**: React Bootstrap
- **Icons**: Font Awesome
- **Storage**: Local Storage
- **Styling**: CSS3 with CSS Variables

## Getting Started

### Prerequisites

- Node.js (v14 or higher)
- npm or yarn

### Installation

1. Clone the repository:
```bash
git clone https://github.com/mujju-11/blinkit-clone.git
cd blinkit-clone
```

2. Install dependencies:
```bash
npm install
# or
yarn install
```

3. Start the development server:
```bash
npm start
# or
yarn start
```

4. Open [http://localhost:3000](http://localhost:3000) to view it in your browser.

## Project Structure

- `/src/components` - Reusable UI components
- `/src/pages` - Page components
- `/src/context` - React context providers
- `/src/services` - Service classes for data handling
- `/src/utils` - Utility functions and constants
- `/src/assets` - Static assets like images

## Runtime Architecture & Memory Design

> **Important:** This repository is a **React web application** (Blinkit e-commerce clone). It contains **no ML model, no on-device inference engine, no mobile hardware delegate (GPU/NPU/NNAPI/Core ML), and no native runtime** of any kind. All prior conversation about `google-ai-edge/LiteRT-LM` or `google-ai-edge/gallery` refers to entirely different, unrelated repositories.

The sections below explain exactly how *this* codebase manages runtime data, memory, and resources — grounded strictly in the source files.

---

### 1. Application Bootstrap & Provider Tree (`src/App.js`)

On startup, `App.js` mounts a nested Context Provider tree:

```
AuthProvider          → src/context/AuthContext.js
  CartProvider        → src/context/CartContext.js
    WishlistProvider  → src/context/WishlistContext.js
      LocationProvider→ src/context/LocationContext.js
        <Router + Pages>
```

Each provider holds its slice of state in **React's in-memory virtual DOM** (JavaScript heap). There is no native memory allocation, no binary buffer, and no hardware pointer involved.

---

### 2. Persistent Storage: `localStorage` (Browser Storage API)

All user data is persisted to the browser's `localStorage` — a synchronous, string-keyed store with a typical 5–10 MB quota per origin. No server, database, or network call is used.

| Key stored | Written by | Read by |
|---|---|---|
| `cart` | `CartContext.js` → `useEffect` on every `cart` state change | `CartContext.js` → first-mount `useEffect` |
| `wishlist` | `WishlistContext.js` → `useEffect` on every `wishlist` state change | `WishlistContext.js` → first-mount `useEffect` |
| `user` | `AuthContext.js` → `login()` / `adminLogin()` | `AuthContext.js` → first-mount `useEffect` |
| `currentLocation` | `LocationContext.js` → `useEffect` on location change | `LocationContext.js` → first-mount `useEffect` |
| `addresses` | `LocationContext.js` → `useEffect` on addresses change | `LocationContext.js` → first-mount `useEffect` |
| `products` | `productService.js` → `initializeData()` (first run only) | `productService.js` → `getAllProducts()` |
| `categories` | `productService.js` → `initializeData()` (first run only) | `productService.js` → `getAllCategories()` |
| `orders` | `orderService.js` → `createOrder()` / `updateOrderStatus()` | `orderService.js` → `getAllOrders()` |

**How the read/write cycle works** (example from `src/context/CartContext.js`):

```js
// Mount: load persisted cart into React state (JS heap)
useEffect(() => {
  const storedCart = localStorage.getItem('cart');   // string read from storage
  if (storedCart) setCart(JSON.parse(storedCart));   // JSON.parse → JS array in heap

}, []);

// Every state change: serialize back to storage
useEffect(() => {
  localStorage.setItem('cart', JSON.stringify(cart)); // JSON.stringify → write string
}, [cart]);
```

The same pattern is repeated verbatim in `WishlistContext.js` and `LocationContext.js`.

---

### 3. In-Memory State: React Context & `useState`

Every piece of live application data lives in a `useState` hook inside a Context provider. React stores these values on the **JavaScript heap** managed by the browser's V8 (or equivalent) engine. There is no manual memory allocation or deallocation.

| Context file | State variable | Type | Cleared when |
|---|---|---|---|
| `AuthContext.js` | `user`, `currentUser` | Object / null | `logout()` sets both to `null`; `localStorage.removeItem('user')` |
| `CartContext.js` | `cart` | Array of product objects | `clearCart()` sets to `[]` |
| `WishlistContext.js` | `wishlist` | Array of product objects | Removed item-by-item via `removeFromWishlist()` |
| `LocationContext.js` | `currentLocation`, `addresses` | String, Array | `deleteAddress()` filters array; location replaced on update |

---

### 4. Product & Order Data (`src/services/`)

#### `productService.js` — `ProductService` class (singleton)

- `initializeData()`: on construction, checks `localStorage` for `products` and `categories`. If absent, serializes the full static `PRODUCTS` and `CATEGORIES` arrays from `src/utils/constants.js` and writes them to storage once.
- `getAllProducts()`: reads and `JSON.parse`s the stored string on *every call* — there is no in-memory cache beyond the caller's own scope.
- `searchProducts(query)`: loads the full product array and runs a `String.prototype.includes` linear scan on `name`, `category`, and `description` fields.

#### `orderService.js` — `OrderService` class (singleton)

- `createOrder(orderData)`: appends a new order object (with a generated ID, ISO date string, and a 5-step timeline array) to the orders array and overwrites `localStorage.orders`.
- `updateOrderStatus(orderId, newStatus)`: re-reads the full orders array, mutates the matching entry in place, and saves the whole array back.
- `getOrderStats()`: reads all orders and performs three `Array.filter` passes to count pending/delivered orders plus a `reduce` for total revenue.

---

### 5. Utility Functions (`src/utils/helpers.js`)

| Function | Purpose | Notes |
|---|---|---|
| `debounce(func, wait)` | Delays execution until `wait` ms after the last call | Used for search input; holds one `setTimeout` handle in closure |
| `throttle(func, limit)` | Limits function execution to once per `limit` ms | Holds one boolean flag in closure |
| `formatCurrency(amount)` | Formats numbers as Indian Rupees | Uses `Intl.NumberFormat` (no allocation beyond format object) |
| `scrollToTop()` | Smooth-scrolls the browser window | Uses `window.scrollTo` |
| `copyToClipboard(text)` | Copies text via `navigator.clipboard` | Async, no persistent state |

---

### 6. Custom Hook: `useProducts` / `useProduct` (`src/hooks/useProducts.js`)

`useProducts()` and `useProduct(id)` wrap `productService` calls inside `useEffect` + `useState`. The product array is loaded from `localStorage` (via `productService.getAllProducts()`) into component-local state when the component mounts. React garbage-collects the state when the component unmounts — no manual cleanup required.

---

### 7. Summary: What Uses "RAM" at Runtime

| Resource | What holds it | Size |
|---|---|---|
| React virtual DOM tree | Browser JS heap | Small (UI nodes only) |
| `cart` state array | `CartContext` → JS heap | One object per cart item (~5–10 fields each) |
| `wishlist` state array | `WishlistContext` → JS heap | One object per saved product |
| `products` array | `productService.getAllProducts()` caller scope | ~100 product objects per call; released after render |
| `orders` array | `orderService.getAllOrders()` caller scope | Grows with user order history |
| `localStorage` data | Browser storage partition (disk-backed) | Total ≤ ~1–2 MB for typical usage |

**There is no GPU usage, no NPU/NNAPI/Core ML delegate, no model file, no tensor buffer, no KV-cache, and no hardware-accelerated inference path anywhere in this repository.** The only hardware resource this application uses beyond normal browser CPU time and JavaScript heap RAM is the browser's `localStorage` quota on the device's storage.

---

## License

MIT