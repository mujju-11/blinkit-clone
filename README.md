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

## Architecture & Resource Handling

See [ARCHITECTURE.md](./ARCHITECTURE.md) for a detailed, code-based explanation of:

- Whether this codebase contains any on-device ML model / inference code (it does not)
- How RAM is allocated and released (React state, garbage collection)
- How persistent storage works (`localStorage` via service classes and context hooks)
- Which browser hardware resources are used and how

## License

MIT