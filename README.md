<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=yes">
  <title>QR Dine | Admin Portal - Complete Restaurant Management</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0-beta3/css/all.min.css">
  <link href="https://fonts.googleapis.com/css2?family=Inter:opsz,wght@14..32,300;400;500;600;700&display=swap" rel="stylesheet">
  <style>
    * { font-family: 'Inter', sans-serif; }
    body { background: #f1f5f9; }
    .admin-card { transition: all 0.2s ease; cursor: pointer; }
    .admin-card:hover { transform: translateY(-2px); box-shadow: 0 8px 20px rgba(0,0,0,0.1); }
    .modal-enter { animation: modalFadeIn 0.2s ease-out; }
    @keyframes modalFadeIn { from { opacity: 0; transform: scale(0.95); } to { opacity: 1; transform: scale(1); } }
    .badge-available { background: #dcfce7; color: #166534; }
    .badge-unavailable { background: #fee2e2; color: #991b1b; }
    .order-row { transition: all 0.2s; cursor: pointer; }
    .order-row:hover { background: #fef3c7; }
    .btn-primary { background: linear-gradient(135deg, #f97316, #ea580c); }
    .btn-primary:hover { background: linear-gradient(135deg, #ea580c, #c2410c); }
    @keyframes slideUp { from { opacity: 0; transform: translateY(20px); } to { opacity: 1; transform: translateY(0); } }
    .animate-slide-up { animation: slideUp 0.2s ease-out; }
    .loading-spinner {
      border: 3px solid #f3f3f3;
      border-top: 3px solid #f97316;
      border-radius: 50%;
      width: 40px;
      height: 40px;
      animation: spin 1s linear infinite;
    }
    @keyframes spin {
      0% { transform: rotate(0deg); }
      100% { transform: rotate(360deg); }
    }
    .file-preview {
      width: 120px;
      height: 120px;
      object-fit: cover;
      border-radius: 8px;
      border: 2px solid #f97316;
    }
    .file-preview-pdf {
      width: 120px;
      height: 120px;
      display: flex;
      align-items: center;
      justify-content: center;
      background: #fef3c7;
      border-radius: 8px;
      border: 2px solid #f97316;
    }
    .upload-progress {
      transition: width 0.3s ease;
    }
    .toast {
      position: fixed;
      top: 20px;
      right: 20px;
      z-index: 9999;
      animation: slideUp 0.3s ease-out;
    }
    .menu-image {
      width: 100%;
      height: 200px;
      object-fit: cover;
      background: #f1f5f9;
    }
  </style>
</head>
<body>
<div id="admin-root"></div>

<script type="module">
  // ======================== FIREBASE CONFIGURATION ========================
  // REPLACE WITH YOUR FIREBASE CONFIG FROM FIREBASE CONSOLE
  const firebaseConfig = {
  apiKey: "AIzaSyDnTl39uYi-NHhBEMeObezN2Uyqkk8sk6Q",
  authDomain: "qrdines-829b7.firebaseapp.com",
  projectId: "qrdines-829b7",
  storageBucket: "qrdines-829b7.firebasestorage.app",
  messagingSenderId: "1025530812948",
  appId: "1:1025530812948:web:66f8e69e0d5af83461e501",
  measurementId: "G-8Y4DWD208G"
};
  // Import Firebase modules
  import { initializeApp } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-app.js";
  import { 
    getFirestore, 
    collection, 
    addDoc, 
    updateDoc, 
    deleteDoc, 
    doc, 
    onSnapshot, 
    query, 
    orderBy, 
    serverTimestamp,
    getDocs,
    where
  } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-firestore.js";
  import {
    getStorage,
    ref,
    uploadBytesResumable,
    getDownloadURL,
    deleteObject
  } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-storage.js";

  // Initialize Firebase
  const app = initializeApp(firebaseConfig);
  const db = getFirestore(app);
  const storage = getStorage(app);
  
  // ======================== TAX/GST CONFIGURATION ========================
  const TAX_CONFIG = {
    cgst: { rate: 2.5, name: "CGST", enabled: true },
    sgst: { rate: 2.5, name: "SGST", enabled: true },
    serviceCharge: { rate: 5, name: "Service Charge", enabled: true },
    roundOff: true
  };
  
  // ======================== GLOBAL STATE ========================
  let menuItems = [];
  let orders = [];
  let isLoggedIn = localStorage.getItem('admin_logged') === 'true';
  let currentView = 'dashboard';
  let currentTab = 'pending';
  let editingItem = null;
  let showModal = false;
  let selectedFile = null;
  let imagePreviewUrl = null;
  let isLoading = true;
  let uploadProgress = 0;
  let isUploading = false;
  let showTaxModal = false;
  
  const categories = ['Starters', 'Main Course', 'Breads', 'Beverages', 'Desserts', 'Soups', 'Salads', 'Seafood', 'Tandoor', 'Rolls', 'Combo Meals'];
  const ADMIN_EMAIL = "admin@resto.com";
  const ADMIN_PASSWORD = "admin123";
  
  // File upload configuration - SUPPORT 0-100MB
  const ALLOWED_FILE_TYPES = [
    'image/jpeg', 'image/png', 'image/gif', 'image/webp', 'image/bmp', 
    'image/svg+xml', 'application/pdf', 'image/jpg'
  ];
  const MAX_FILE_SIZE = 100 * 1024 * 1024; // 100MB
  
  // ======================== HELPER FUNCTIONS ========================
  function showNotification(message, type = 'success') {
    const toast = document.createElement('div');
    const colors = { 
      success: '#22c55e', 
      error: '#ef4444', 
      info: '#3b82f6',
      warning: '#f59e0b'
    };
    const icons = {
      success: 'fa-check-circle',
      error: 'fa-exclamation-circle',
      info: 'fa-info-circle',
      warning: 'fa-exclamation-triangle'
    };
    
    toast.className = `toast px-5 py-3 rounded-lg shadow-lg text-white font-medium text-sm flex items-center gap-2`;
    toast.style.background = colors[type] || colors.success;
    toast.innerHTML = `<i class="fas ${icons[type]} mr-2"></i>${message}`;
    document.body.appendChild(toast);
    setTimeout(() => toast.remove(), 3000);
  }
  
  function formatCurrency(amount) {
    return new Intl.NumberFormat('en-IN', { style: 'currency', currency: 'INR' }).format(amount);
  }
  
  function validateFile(file) {
    if (!file) return { valid: false, message: "No file selected" };
    
    if (!ALLOWED_FILE_TYPES.includes(file.type)) {
      return { valid: false, message: `Invalid file type. Allowed: JPG, PNG, GIF, WEBP, PDF (${file.type})` };
    }
    
    if (file.size > MAX_FILE_SIZE) {
      return { valid: false, message: `File too large. Max size: 100MB (Current: ${(file.size / (1024 * 1024)).toFixed(2)}MB)` };
    }
    
    return { valid: true, message: "File valid" };
  }
  
  // ======================== FILE UPLOAD FUNCTIONS ========================
  async function uploadFile(file, itemId) {
    return new Promise((resolve, reject) => {
      const validation = validateFile(file);
      if (!validation.valid) {
        reject(validation.message);
        return;
      }
      
      isUploading = true;
      uploadProgress = 0;
      render();
      
      const fileExtension = file.name.split('.').pop();
      const fileName = `menu_items/${itemId}_${Date.now()}.${fileExtension}`;
      const storageRef = ref(storage, fileName);
      const uploadTask = uploadBytesResumable(storageRef, file);
      
      uploadTask.on('state_changed',
        (snapshot) => {
          uploadProgress = (snapshot.bytesTransferred / snapshot.totalBytes) * 100;
          render();
        },
        (error) => {
          console.error("Upload error:", error);
          isUploading = false;
          reject(error.message);
        },
        async () => {
          const downloadUrl = await getDownloadURL(uploadTask.snapshot.ref);
          isUploading = false;
          uploadProgress = 0;
          resolve(downloadUrl);
        }
      );
    });
  }
  
  async function deleteFile(fileUrl) {
    if (!fileUrl || !fileUrl.includes('firebasestorage')) return;
    
    try {
      const fileRef = ref(storage, fileUrl);
      await deleteObject(fileRef);
      console.log("File deleted successfully");
    } catch (error) {
      console.error("Error deleting file:", error);
    }
  }
  
  // ======================== TAX CALCULATION ========================
  function calculateTax(subtotal) {
    let cgst = 0, sgst = 0, serviceCharge = 0, totalTax = 0;
    
    if (TAX_CONFIG.cgst.enabled) {
      cgst = (subtotal * TAX_CONFIG.cgst.rate) / 100;
    }
    if (TAX_CONFIG.sgst.enabled) {
      sgst = (subtotal * TAX_CONFIG.sgst.rate) / 100;
    }
    totalTax = cgst + sgst;
    
    if (TAX_CONFIG.serviceCharge.enabled) {
      serviceCharge = (subtotal * TAX_CONFIG.serviceCharge.rate) / 100;
      totalTax += serviceCharge;
    }
    
    let grandTotal = subtotal + totalTax;
    let roundOff = 0;
    
    if (TAX_CONFIG.roundOff) {
      const rounded = Math.round(grandTotal);
      roundOff = rounded - grandTotal;
      grandTotal = rounded;
    }
    
    return { subtotal, cgst, sgst, serviceCharge, totalTax, roundOff, grandTotal };
  }
  
  // ======================== FIREBASE CRUD OPERATIONS ========================
  
  // Menu Functions
  function listenToMenu() {
    isLoading = true;
    render();
    
    const menuRef = collection(db, "menu");
    const menuQuery = query(menuRef, orderBy("createdAt", "desc"));
    
    onSnapshot(menuQuery, (snapshot) => {
      menuItems = [];
      snapshot.forEach((doc) => {
        menuItems.push({ id: doc.id, ...doc.data() });
      });
      isLoading = false;
      if (currentView === 'menu') render();
    }, (error) => {
      console.error("Error loading menu:", error);
      isLoading = false;
      showNotification("Error loading menu: " + error.message, "error");
    });
  }
  
  async function addMenuItem(itemData, file) {
    try {
      let fileUrl = "https://placehold.co/400x200/e2e8f0/475569?text=No+Image";
      
      const docRef = await addDoc(collection(db, "menu"), {
        name: itemData.name,
        price: itemData.price,
        category: itemData.category,
        description: itemData.description,
        image: fileUrl,
        available: true,
        createdAt: serverTimestamp(),
        updatedAt: serverTimestamp()
      });
      
      if (file) {
        fileUrl = await uploadFile(file, docRef.id);
        await updateDoc(docRef, { image: fileUrl });
      }
      
      showNotification(`${itemData.name} added successfully!`, 'success');
      return true;
    } catch (error) {
      console.error("Error adding item:", error);
      showNotification("Error adding item: " + error.message, 'error');
      return false;
    }
  }
  
  async function updateMenuItem(id, updates, newFile = null) {
    try {
      let fileUrl = updates.image;
      
      if (newFile) {
        const oldItem = menuItems.find(i => i.id === id);
        if (oldItem && oldItem.image && oldItem.image.includes('firebasestorage')) {
          await deleteFile(oldItem.image);
        }
        fileUrl = await uploadFile(newFile, id);
      }
      
      await updateDoc(doc(db, "menu", id), {
        name: updates.name,
        price: updates.price,
        category: updates.category,
        description: updates.description,
        image: fileUrl,
        updatedAt: serverTimestamp()
      });
      
      showNotification('Item updated successfully!', 'success');
      return true;
    } catch (error) {
      console.error("Error updating item:", error);
      showNotification("Error updating item: " + error.message, 'error');
      return false;
    }
  }
  
  async function deleteMenuItem(id, name, imageUrl) {
    if (!confirm(`⚠️ Delete "${name}" permanently?\n\nThis action cannot be undone!`)) return;
    
    try {
      if (imageUrl && imageUrl.includes('firebasestorage')) {
        await deleteFile(imageUrl);
      }
      
      await deleteDoc(doc(db, "menu", id));
      showNotification(`"${name}" deleted successfully!`, 'success');
    } catch (error) {
      console.error("Error deleting item:", error);
      showNotification("Error deleting item: " + error.message, 'error');
    }
  }
  
  async function toggleAvailability(id, currentStatus) {
    try {
      await updateDoc(doc(db, "menu", id), { 
        available: !currentStatus,
        updatedAt: serverTimestamp()
      });
      showNotification(`Item ${!currentStatus ? 'available' : 'unavailable'}`, 'info');
    } catch (error) {
      console.error("Error toggling status:", error);
      showNotification("Error updating status", 'error');
    }
  }
  
  // Order Functions
  function listenToOrders() {
    const ordersRef = collection(db, "orders");
    const ordersQuery = query(ordersRef, orderBy("timestamp", "desc"));
    
    onSnapshot(ordersQuery, (snapshot) => {
      orders = [];
      snapshot.forEach((doc) => {
        orders.push({ id: doc.id, ...doc.data() });
      });
      if (currentView === 'dashboard' || currentView === 'orders') render();
    }, (error) => {
      console.error("Error loading orders:", error);
    });
  }
  
  async function updateOrderStatus(orderId, newStatus) {
    try {
      await updateDoc(doc(db, "orders", orderId), {
        status: newStatus,
        updatedAt: serverTimestamp()
      });
      showNotification(`Order status updated to ${newStatus}`, 'success');
    } catch (error) {
      console.error("Error updating order:", error);
      showNotification("Error updating order", 'error');
    }
  }
  
  // ======================== STATISTICS ========================
  function getStats() {
    const totalOrders = orders.length;
    const pendingOrders = orders.filter(o => o.status === 'Pending').length;
    const preparingOrders = orders.filter(o => o.status === 'Preparing').length;
    const servedOrders = orders.filter(o => o.status === 'Served').length;
    const totalRevenue = orders.reduce((sum, o) => sum + (o.totalPrice || 0), 0);
    const availableItems = menuItems.filter(i => i.available).length;
    const unavailableItems = menuItems.filter(i => !i.available).length;
    
    const today = new Date().toDateString();
    const todayRevenue = orders
      .filter(o => new Date(o.timestamp).toDateString() === today)
      .reduce((sum, o) => sum + (o.totalPrice || 0), 0);
    
    return { 
      totalOrders, pendingOrders, preparingOrders, servedOrders, 
      totalRevenue, availableItems, unavailableItems, todayRevenue 
    };
  }
  
  // ======================== UI FUNCTIONS ========================
  function handleFileSelect(event) {
    const file = event.target.files[0];
    if (file) {
      const validation = validateFile(file);
      if (!validation.valid) {
        showNotification(validation.message, 'error');
        event.target.value = '';
        return;
      }
      
      selectedFile = file;
      const reader = new FileReader();
      reader.onload = (e) => {
        imagePreviewUrl = e.target.result;
        const preview = document.getElementById('filePreview');
        if (preview) {
          if (file.type === 'application/pdf') {
            preview.src = 'https://cdn-icons-png.flaticon.com/512/337/337946.png';
          } else {
            preview.src = imagePreviewUrl;
          }
          preview.classList.remove('hidden');
        }
        const fileInfo = document.getElementById('fileInfo');
        if (fileInfo) {
          fileInfo.innerHTML = `<i class="fas fa-check-circle text-green-500"></i> ${file.name} (${(file.size / (1024 * 1024)).toFixed(2)} MB)`;
          fileInfo.classList.remove('hidden');
        }
      };
      reader.readAsDataURL(file);
    }
  }
  
  function escapeHtml(str) {
    if (!str) return '';
    return str.replace(/[&<>]/g, function(m) {
      if (m === '&') return '&amp;';
      if (m === '<') return '&lt;';
      if (m === '>') return '&gt;';
      return m;
    });
  }
  
  // ======================== RENDER FUNCTIONS ========================
  function renderLogin() {
    return `
      <div class="min-h-screen flex items-center justify-center bg-gradient-to-br from-orange-50 to-amber-100">
        <div class="bg-white rounded-2xl p-8 max-w-md w-full shadow-2xl">
          <div class="text-center mb-6">
            <div class="inline-block bg-orange-100 rounded-full p-4 mb-4">
              <i class="fas fa-utensils text-5xl text-orange-600"></i>
            </div>
            <h2 class="text-2xl font-bold text-gray-800">Admin Portal</h2>
            <p class="text-gray-500 text-sm mt-1">Restaurant Management System</p>
          </div>
          <div class="space-y-4">
            <input type="email" id="email" class="w-full border border-gray-300 rounded-lg px-4 py-2 focus:ring-2 focus:ring-orange-500" placeholder="admin@resto.com" value="admin@resto.com">
            <input type="password" id="password" class="w-full border border-gray-300 rounded-lg px-4 py-2 focus:ring-2 focus:ring-orange-500" placeholder="admin123" value="admin123">
            <button id="loginBtn" class="w-full bg-orange-600 text-white py-2 rounded-lg font-semibold hover:bg-orange-700 transition">
              <i class="fas fa-sign-in-alt mr-2"></i> Login
            </button>
          </div>
          <p class="text-xs text-center text-gray-400 mt-6">Demo: admin@resto.com / admin123</p>
        </div>
      </div>
    `;
  }
  
  function renderDashboard() {
    const stats = getStats();
    const recentOrders = orders.slice(0, 5);
    
    return `
      <div class="p-6">
        <div class="bg-white rounded-2xl shadow-sm p-6 mb-6 flex flex-wrap justify-between items-center gap-4">
          <div>
            <h1 class="text-2xl font-bold text-gray-800">Dashboard</h1>
            <p class="text-gray-500 text-sm mt-1">Live Restaurant Management System</p>
          </div>
          <div class="flex gap-3">
            <button id="taxSettingsBtn" class="bg-purple-500 text-white px-4 py-2 rounded-lg hover:bg-purple-600 transition">
              <i class="fas fa-percent mr-2"></i> Tax Settings
            </button>
            <button id="logoutBtn" class="bg-red-500 text-white px-4 py-2 rounded-lg hover:bg-red-600 transition">
              <i class="fas fa-sign-out-alt mr-2"></i> Logout
            </button>
          </div>
        </div>
        
        <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-5 mb-6">
          <div class="bg-white rounded-xl shadow-sm p-5 admin-card">
            <div class="flex justify-between items-start">
              <div><p class="text-gray-500 text-sm">Total Orders</p><p class="text-3xl font-bold text-gray-800">${stats.totalOrders}</p></div>
              <i class="fas fa-receipt text-4xl text-orange-300"></i>
            </div>
          </div>
          <div class="bg-white rounded-xl shadow-sm p-5 admin-card">
            <div class="flex justify-between items-start">
              <div><p class="text-gray-500 text-sm">Pending Orders</p><p class="text-3xl font-bold text-yellow-600">${stats.pendingOrders}</p></div>
              <i class="fas fa-clock text-4xl text-yellow-300"></i>
            </div>
          </div>
          <div class="bg-white rounded-xl shadow-sm p-5 admin-card">
            <div class="flex justify-between items-start">
              <div><p class="text-gray-500 text-sm">Total Revenue</p><p class="text-3xl font-bold text-green-600">${formatCurrency(stats.totalRevenue)}</p></div>
              <i class="fas fa-rupee-sign text-4xl text-green-300"></i>
            </div>
          </div>
          <div class="bg-white rounded-xl shadow-sm p-5 admin-card">
            <div class="flex justify-between items-start">
              <div><p class="text-gray-500 text-sm">Active Menu</p><p class="text-3xl font-bold text-orange-600">${stats.availableItems}/${stats.availableItems+stats.unavailableItems}</p></div>
              <i class="fas fa-utensils text-4xl text-orange-300"></i>
            </div>
          </div>
        </div>
        
        <div class="grid grid-cols-1 md:grid-cols-2 gap-6 mb-6">
          <div class="bg-gradient-to-r from-orange-500 to-orange-600 rounded-xl shadow-md p-6 text-white cursor-pointer hover:shadow-xl transition" id="goToMenuBtn">
            <h3 class="text-xl font-bold mb-2">🍽️ Menu Management</h3>
            <p class="text-orange-100 text-sm mb-4">Add/Edit/Delete menu items with file upload (up to 100MB)</p>
            <div class="inline-block bg-white text-orange-600 px-4 py-2 rounded-lg font-semibold"><i class="fas fa-edit mr-2"></i> Manage Menu</div>
          </div>
          <div class="bg-gradient-to-r from-blue-500 to-blue-600 rounded-xl shadow-md p-6 text-white cursor-pointer hover:shadow-xl transition" id="goToOrdersBtn">
            <h3 class="text-xl font-bold mb-2">📦 Order Tracking</h3>
            <p class="text-blue-100 text-sm mb-4">Track and manage orders in real-time</p>
            <div class="inline-block bg-white text-blue-600 px-4 py-2 rounded-lg font-semibold"><i class="fas fa-truck-fast mr-2"></i> View Orders</div>
          </div>
        </div>
        
        <div class="bg-white rounded-xl shadow-sm p-6">
          <div class="flex justify-between items-center mb-4">
            <h2 class="text-lg font-bold text-gray-800">📋 Recent Orders</h2>
            <button id="viewAllOrdersBtn" class="text-orange-600 text-sm hover:underline">View All →</button>
          </div>
          ${recentOrders.length === 0 ? `
            <div class="text-center py-8">
              <i class="fas fa-shopping-bag text-5xl text-gray-300 mb-3"></i>
              <p class="text-gray-500">No orders yet</p>
            </div>
          ` : `
            <div class="space-y-3 max-h-96 overflow-y-auto">
              ${recentOrders.map(order => `
                <div class="order-row border rounded-lg p-4 flex flex-wrap justify-between items-center gap-3 hover:bg-gray-50">
                  <div>
                    <p class="font-semibold text-gray-800">#${order.id.slice(-8)}</p>
                    <p class="text-sm text-gray-500">Table ${order.tableNumber} • ${new Date(order.timestamp).toLocaleTimeString()}</p>
                  </div>
                  <div>
                    <p class="font-medium">${order.items?.length || 0} items</p>
                    <p class="text-orange-600 font-bold">${formatCurrency(order.totalPrice)}</p>
                  </div>
                  <div>
                    <span class="px-3 py-1 rounded-full text-xs font-semibold ${order.status === 'Pending' ? 'bg-yellow-100 text-yellow-700' : order.status === 'Preparing' ? 'bg-blue-100 text-blue-700' : 'bg-green-100 text-green-700'}">
                      ${order.status}
                    </span>
                  </div>
                  <div class="flex gap-2">
                    ${order.status === 'Pending' ? `<button class="acceptOrderBtn bg-green-500 text-white px-3 py-1 rounded text-xs hover:bg-green-600" data-id="${order.id}">Accept</button>` : ''}
                    ${order.status === 'Preparing' ? `<button class="serveOrderBtn bg-purple-500 text-white px-3 py-1 rounded text-xs hover:bg-purple-600" data-id="${order.id}">Serve</button>` : ''}
                  </div>
                </div>
              `).join('')}
            </div>
          `}
        </div>
      </div>
    `;
  }
  
  function renderMenuManagement() {
    if (isLoading) {
      return `
        <div class="min-h-screen flex items-center justify-center">
          <div class="text-center">
            <div class="loading-spinner mx-auto"></div>
            <p class="mt-4 text-gray-600">Loading menu items...</p>
          </div>
        </div>
      `;
    }
    
    const stats = getStats();
    
    return `
      <div class="p-6">
        <div class="bg-white rounded-2xl shadow-sm p-6 mb-6 flex flex-wrap justify-between items-center gap-4">
          <div>
            <button id="backToDashboard" class="text-gray-600 hover:text-orange-600 mr-4"><i class="fas fa-arrow-left"></i> Back</button>
            <h1 class="text-2xl font-bold text-gray-800 inline-block">Menu Management</h1>
            <p class="text-gray-500 text-sm mt-1">${stats.availableItems} available • ${stats.unavailableItems} unavailable</p>
          </div>
          <button id="addItemBtn" class="bg-orange-600 text-white px-5 py-2 rounded-lg font-semibold hover:bg-orange-700 transition">
            <i class="fas fa-plus mr-2"></i> Add New Item
          </button>
        </div>
        
        <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-5">
          ${menuItems.map(item => `
            <div class="bg-white rounded-xl shadow-sm overflow-hidden admin-card">
              <div class="relative h-48 bg-gray-100">
                <img src="${item.image}" class="menu-image" alt="${escapeHtml(item.name)}" 
                  onerror="this.src='https://placehold.co/400x200/e2e8f0/475569?text=No+Image'">
                <div class="absolute top-2 right-2">
                  <span class="px-2 py-1 rounded-full text-xs font-semibold ${item.available ? 'badge-available' : 'badge-unavailable'}">
                    ${item.available ? 'Available' : 'Unavailable'}
                  </span>
                </div>
              </div>
              <div class="p-4">
                <div class="flex justify-between items-start mb-2">
                  <div>
                    <h3 class="font-bold text-gray-800 text-lg">${escapeHtml(item.name)}</h3>
                    <p class="text-xs text-gray-500">${item.category}</p>
                  </div>
                  <p class="text-orange-600 font-bold text-xl">${formatCurrency(item.price)}</p>
                </div>
                <p class="text-gray-500 text-sm mb-4 line-clamp-2">${escapeHtml(item.description || 'No description')}</p>
                <div class="flex gap-2">
                  <button class="editItemBtn flex-1 bg-gray-100 text-gray-700 px-3 py-2 rounded-lg text-sm hover:bg-gray-200 transition" data-id="${item.id}">
                    <i class="fas fa-edit mr-1"></i> Edit
                  </button>
                  <button class="toggleAvailBtn flex-1 ${item.available ? 'bg-yellow-100 text-yellow-700' : 'bg-green-100 text-green-700'} px-3 py-2 rounded-lg text-sm hover:opacity-80 transition" data-id="${item.id}" data-available="${item.available}">
                    <i class="fas ${item.available ? 'fa-ban' : 'fa-check'} mr-1"></i> ${item.available ? 'Block' : 'Unblock'}
                  </button>
                  <button class="deleteItemBtn bg-red-100 text-red-600 px-3 py-2 rounded-lg text-sm hover:bg-red-200 transition" data-id="${item.id}" data-name="${escapeHtml(item.name)}" data-image="${item.image}">
                    <i class="fas fa-trash"></i>
                  </button>
                </div>
                ${item.image && item.image.includes('firebasestorage') ? '<p class="text-xs text-green-500 mt-2"><i class="fas fa-cloud-upload-alt"></i> Image uploaded</p>' : ''}
              </div>
            </div>
          `).join('')}
        </div>
      </div>
    `;
  }
  
  function renderOrdersManagement() {
    const pending = orders.filter(o => o.status === 'Pending');
    const preparing = orders.filter(o => o.status === 'Preparing');
    const completed = orders.filter(o => o.status === 'Served');
    let ordersToShow = [];
    
    if (currentTab === 'pending') ordersToShow = pending;
    else if (currentTab === 'preparing') ordersToShow = preparing;
    else ordersToShow = completed;
    
    return `
      <div class="p-6">
        <div class="bg-white rounded-2xl shadow-sm p-6 mb-6">
          <button id="backToDashboardOrders" class="text-gray-600 hover:text-orange-600 mr-4"><i class="fas fa-arrow-left"></i> Back</button>
          <h1 class="text-2xl font-bold text-gray-800 inline-block">Order Management</h1>
          <p class="text-gray-500 text-sm mt-1">Real-time order tracking</p>
        </div>
        
        <div class="flex gap-2 mb-6 border-b">
          <button class="order-tab px-5 py-2 font-medium transition ${currentTab === 'pending' ? 'text-orange-600 border-b-2 border-orange-600' : 'text-gray-500 hover:text-orange-600'}" data-tab="pending">
            Pending (${pending.length})
          </button>
          <button class="order-tab px-5 py-2 font-medium transition ${currentTab === 'preparing' ? 'text-orange-600 border-b-2 border-orange-600' : 'text-gray-500 hover:text-orange-600'}" data-tab="preparing">
            Preparing (${preparing.length})
          </button>
          <button class="order-tab px-5 py-2 font-medium transition ${currentTab === 'completed' ? 'text-orange-600 border-b-2 border-orange-600' : 'text-gray-500 hover:text-orange-600'}" data-tab="completed">
            Completed (${completed.length})
          </button>
        </div>
        
        <div class="space-y-4">
          ${ordersToShow.length === 0 ? `
            <div class="bg-white rounded-xl p-12 text-center">
              <i class="fas fa-inbox text-5xl text-gray-300 mb-3"></i>
              <p class="text-gray-500">No orders in this category</p>
            </div>
          ` : ordersToShow.map(order => `
            <div class="bg-white rounded-xl shadow-sm p-5 order-row">
              <div class="flex flex-wrap justify-between items-start gap-3 mb-4">
                <div>
                  <h3 class="font-bold text-gray-800">Order #${order.id.slice(-8)}</h3>
                  <p class="text-sm text-gray-500">Table ${order.tableNumber} • ${new Date(order.timestamp).toLocaleString()}</p>
                </div>
                <div class="text-right">
                  <p class="text-2xl font-bold text-orange-600">${formatCurrency(order.totalPrice)}</p>
                  <span class="px-3 py-1 rounded-full text-xs font-semibold ${order.status === 'Pending' ? 'bg-yellow-100 text-yellow-700' : order.status === 'Preparing' ? 'bg-blue-100 text-blue-700' : 'bg-green-100 text-green-700'}">
                    ${order.status}
                  </span>
                </div>
              </div>
              <div class="border-t pt-3 mb-3">
                <p class="font-medium text-gray-700 mb-2">Order Items:</p>
                <div class="space-y-1">
                  ${order.items.map(item => `
                    <div class="flex justify-between text-sm">
                      <span>${escapeHtml(item.name)} <span class="text-gray-500">x${item.quantity}</span></span>
                      <span>${formatCurrency(item.price * item.quantity)}</span>
                    </div>
                  `).join('')}
                </div>
              </div>
              <div class="flex gap-2 pt-2">
                ${order.status === 'Pending' ? `
                  <button class="acceptOrderBtn bg-green-500 text-white px-4 py-2 rounded-lg text-sm font-medium hover:bg-green-600 transition" data-id="${order.id}">
                    <i class="fas fa-check mr-1"></i> Accept & Prepare
                  </button>
                ` : ''}
                ${order.status === 'Preparing' ? `
                  <button class="serveOrderBtn bg-purple-500 text-white px-4 py-2 rounded-lg text-sm font-medium hover:bg-purple-600 transition" data-id="${order.id}">
                    <i class="fas fa-check-double mr-1"></i> Mark as Served
                  </button>
                ` : ''}
              </div>
            </div>
          `).join('')}
        </div>
      </div>
    `;
  }
  
  function renderTaxModal() {
    if (!showTaxModal) return '';
    
    return `
      <div class="fixed inset-0 bg-black/50 flex items-center justify-center z-50 p-4" id="taxModalOverlay">
        <div class="bg-white rounded-2xl max-w-md w-full p-6 modal-enter">
          <div class="flex justify-between items-center mb-4">
            <h2 class="text-xl font-bold text-gray-800"><i class="fas fa-percent text-orange-600 mr-2"></i> Tax Settings</h2>
            <button id="closeTaxModal" class="text-gray-400 hover:text-gray-600 text-2xl">&times;</button>
          </div>
          
          <div class="space-y-4">
            <div class="border-b pb-3">
              <h3 class="font-semibold text-gray-700 mb-2">GST Settings</h3>
              <label class="flex items-center justify-between py-2">
                <span class="text-sm">CGST (2.5%)</span>
                <input type="checkbox" id="cgstEnabled" ${TAX_CONFIG.cgst.enabled ? 'checked' : ''} class="toggle-tax w-4 h-4 text-orange-600">
              </label>
              <label class="flex items-center justify-between py-2">
                <span class="text-sm">SGST (2.5%)</span>
                <input type="checkbox" id="sgstEnabled" ${TAX_CONFIG.sgst.enabled ? 'checked' : ''} class="toggle-tax w-4 h-4 text-orange-600">
              </label>
            </div>
            
            <div class="border-b pb-3">
              <h3 class="font-semibold text-gray-700 mb-2">Additional Charges</h3>
              <label class="flex items-center justify-between py-2">
                <span class="text-sm">Service Charge (5%)</span>
                <input type="checkbox" id="serviceChargeEnabled" ${TAX_CONFIG.serviceCharge.enabled ? 'checked' : ''} class="w-4 h-4 text-orange-600">
              </label>
            </div>
            
            <div>
              <label class="flex items-center justify-between py-2">
                <span class="text-sm">Round Off to Nearest Rupee</span>
                <input type="checkbox" id="roundOffEnabled" ${TAX_CONFIG.roundOff ? 'checked' : ''} class="w-4 h-4 text-orange-600">
              </label>
            </div>
          </div>
          
          <div class="flex gap-3 mt-6">
            <button id="saveTaxBtn" class="flex-1 bg-orange-600 text-white py-2 rounded-lg font-semibold hover:bg-orange-700 transition">
              <i class="fas fa-save mr-2"></i> Save Settings
            </button>
            <button id="cancelTaxBtn" class="flex-1 bg-gray-100 text-gray-700 py-2 rounded-lg font-semibold hover:bg-gray-200 transition">
              Cancel
            </button>
          </div>
        </div>
      </div>
    `;
  }
  
  function renderAddEditModal() {
    if (!showModal) return '';
    const isEditing = editingItem !== null;
    const item = editingItem || {};
    
    return `
      <div class="fixed inset-0 bg-black/50 flex items-center justify-center z-50 p-4" id="modalOverlay">
        <div class="bg-white rounded-2xl max-w-md w-full p-6 max-h-[90vh] overflow-y-auto modal-enter">
          <div class="flex justify-between items-center mb-4">
            <h2 class="text-xl font-bold text-gray-800">${isEditing ? 'Edit Item' : 'Add New Item'}</h2>
            <button id="closeModalBtn" class="text-gray-400 hover:text-gray-600 text-2xl">&times;</button>
          </div>
          
          <div class="space-y-4">
            <div>
              <label class="block text-sm font-medium text-gray-700 mb-1">Item Name *</label>
              <input type="text" id="itemName" class="w-full border border-gray-300 rounded-lg px-3 py-2 focus:ring-2 focus:ring-orange-500" placeholder="e.g., Butter Chicken" value="${escapeHtml(item.name || '')}">
            </div>
            
            <div>
              <label class="block text-sm font-medium text-gray-700 mb-1">Price (₹) *</label>
              <input type="number" id="itemPrice" class="w-full border border-gray-300 rounded-lg px-3 py-2 focus:ring-2 focus:ring-orange-500" placeholder="299" value="${item.price || ''}">
            </div>
            
            <div>
              <label class="block text-sm font-medium text-gray-700 mb-1">Category</label>
              <select id="itemCategory" class="w-full border border-gray-300 rounded-lg px-3 py-2 focus:ring-2 focus:ring-orange-500">
                ${categories.map(c => `<option value="${c}" ${item.category === c ? 'selected' : ''}>${c}</option>`).join('')}
              </select>
            </div>
            
            <div>
              <label class="block text-sm font-medium text-gray-700 mb-1">Description</label>
              <textarea id="itemDescription" class="w-full border border-gray-300 rounded-lg px-3 py-2 focus:ring-2 focus:ring-orange-500" rows="3" placeholder="Brief description">${escapeHtml(item.description || '')}</textarea>
            </div>
            
            <div>
              <label class="block text-sm font-medium text-gray-700 mb-1">Upload File (Image/PDF)</label>
              <input type="file" id="itemFile" accept="image/*,application/pdf" class="w-full border border-gray-300 rounded-lg px-3 py-2 focus:ring-2 focus:ring-orange-500">
              <div class="mt-2">
                <img id="filePreview" src="${item.image || ''}" class="file-preview ${item.image ? '' : 'hidden'}" alt="Preview" onerror="this.src='https://placehold.co/120x120/e2e8f0/475569?text=No+Image'">
                <div id="fileInfo" class="text-xs text-gray-500 mt-1 ${!item.image ? 'hidden' : ''}"></div>
              </div>
              ${isUploading ? `
                <div class="mt-2">
                  <div class="bg-gray-200 rounded-full h-2 overflow-hidden">
                    <div class="bg-orange-600 h-2 rounded-full upload-progress" style="width: ${uploadProgress}%"></div>
                  </div>
                  <p class="text-xs text-gray-500 mt-1">Uploading: ${uploadProgress.toFixed(0)}%</p>
                </div>
              ` : ''}
              <p class="text-xs text-gray-400 mt-2">
                <i class="fas fa-info-circle"></i> Supported: JPG, PNG, GIF, WEBP, PDF (Max 100MB)
              </p>
            </div>
          </div>
          
          <div class="flex gap-3 mt-6">
            <button id="saveItemBtn" class="flex-1 bg-orange-600 text-white py-2 rounded-lg font-semibold hover:bg-orange-700 transition" ${isUploading ? 'disabled' : ''}>
              <i class="fas fa-save mr-2"></i> ${isEditing ? 'Update' : 'Add'}
            </button>
            <button id="cancelModalBtn" class="flex-1 bg-gray-100 text-gray-700 py-2 rounded-lg font-semibold hover:bg-gray-200 transition">
              Cancel
            </button>
          </div>
        </div>
      </div>
    `;
  }
  
  function render() {
    const root = document.getElementById('admin-root');
    if (!root) return;
    
    if (!isLoggedIn) {
      root.innerHTML = renderLogin();
      attachLoginEvents();
      return;
    }
    
    let main = '';
    if (currentView === 'dashboard') main = renderDashboard();
    else if (currentView === 'menu') main = renderMenuManagement();
    else if (currentView === 'orders') main = renderOrdersManagement();
    
    root.innerHTML = main + renderAddEditModal() + renderTaxModal();
    attachEvents();
  }
  
  // ======================== EVENT HANDLERS ========================
  function attachLoginEvents() {
    const loginBtn = document.getElementById('loginBtn');
    if (loginBtn) {
      loginBtn.addEventListener('click', () => {
        const email = document.getElementById('email')?.value;
        const pwd = document.getElementById('password')?.value;
        if (email === ADMIN_EMAIL && pwd === ADMIN_PASSWORD) {
          isLoggedIn = true;
          localStorage.setItem('admin_logged', 'true');
          listenToMenu();
          listenToOrders();
          currentView = 'dashboard';
          render();
        } else {
          showNotification('Invalid credentials! Use admin@resto.com / admin123', 'error');
        }
      });
    }
  }
  
  function attachEvents() {
    // Navigation
    document.getElementById('logoutBtn')?.addEventListener('click', () => {
      isLoggedIn = false;
      localStorage.removeItem('admin_logged');
      render();
    });
    
    document.getElementById('goToMenuBtn')?.addEventListener('click', () => {
      currentView = 'menu';
      render();
    });
    
    document.getElementById('goToOrdersBtn')?.addEventListener('click', () => {
      currentView = 'orders';
      currentTab = 'pending';
      render();
    });
    
    document.getElementById('viewAllOrdersBtn')?.addEventListener('click', () => {
      currentView = 'orders';
      currentTab = 'pending';
      render();
    });
    
    document.getElementById('backToDashboard')?.addEventListener('click', () => {
      currentView = 'dashboard';
      render();
    });
    
    document.getElementById('backToDashboardOrders')?.addEventListener('click', () => {
      currentView = 'dashboard';
      render();
    });
    
    document.getElementById('addItemBtn')?.addEventListener('click', () => {
      editingItem = null;
      selectedFile = null;
      imagePreviewUrl = null;
      showModal = true;
      render();
    });
    
    // Tax settings
    document.getElementById('taxSettingsBtn')?.addEventListener('click', () => {
      showTaxModal = true;
      render();
    });
    
    // File upload handler
    const fileInput = document.getElementById('itemFile');
    if (fileInput) {
      fileInput.addEventListener('change', handleFileSelect);
    }
    
    // Menu actions
    document.querySelectorAll('.editItemBtn').forEach(btn => {
      btn.addEventListener('click', () => {
        editingItem = menuItems.find(i => i.id === btn.dataset.id);
        selectedFile = null;
        imagePreviewUrl = editingItem?.image || null;
        showModal = true;
        render();
      });
    });
    
    document.querySelectorAll('.toggleAvailBtn').forEach(btn => {
      btn.addEventListener('click', () => {
        toggleAvailability(btn.dataset.id, btn.dataset.available === 'true');
      });
    });
    
    document.querySelectorAll('.deleteItemBtn').forEach(btn => {
      btn.addEventListener('click', () => {
        deleteMenuItem(btn.dataset.id, btn.dataset.name, btn.dataset.image);
      });
    });
    
    // Order actions
    document.querySelectorAll('.acceptOrderBtn').forEach(btn => {
      btn.addEventListener('click', () => {
        updateOrderStatus(btn.dataset.id, 'Preparing');
      });
    });
    
    document.querySelectorAll('.serveOrderBtn').forEach(btn => {
      btn.addEventListener('click', () => {
        updateOrderStatus(btn.dataset.id, 'Served');
      });
    });
    
    document.querySelectorAll('.order-tab').forEach(tab => {
      tab.addEventListener('click', () => {
        currentTab = tab.dataset.tab;
        render();
      });
    });
    
    // Modal handlers
    const closeModal = () => {
      showModal = false;
      editingItem = null;
      selectedFile = null;
      imagePreviewUrl = null;
      uploadProgress = 0;
      isUploading = false;
      render();
    };
    
    const closeTaxModal = () => {
      showTaxModal = false;
      render();
    };
    
    document.getElementById('closeModalBtn')?.addEventListener('click', closeModal);
    document.getElementById('cancelModalBtn')?.addEventListener('click', closeModal);
    document.getElementById('modalOverlay')?.addEventListener('click', (e) => {
      if (e.target.id === 'modalOverlay') closeModal();
    });
    
    document.getElementById('closeTaxModal')?.addEventListener('click', closeTaxModal);
    document.getElementById('cancelTaxBtn')?.addEventListener('click', closeTaxModal);
    document.getElementById('taxModalOverlay')?.addEventListener('click', (e) => {
      if (e.target.id === 'taxModalOverlay') closeTaxModal();
    });
    
    document.getElementById('saveTaxBtn')?.addEventListener('click', () => {
      TAX_CONFIG.cgst.enabled = document.getElementById('cgstEnabled')?.checked || false;
      TAX_CONFIG.sgst.enabled = document.getElementById('sgstEnabled')?.checked || false;
      TAX_CONFIG.serviceCharge.enabled = document.getElementById('serviceChargeEnabled')?.checked || false;
      TAX_CONFIG.roundOff = document.getElementById('roundOffEnabled')?.checked || false;
      
      showNotification('Tax settings saved successfully!', 'success');
      showTaxModal = false;
      render();
    });
    
    document.getElementById('saveItemBtn')?.addEventListener('click', async () => {
      const name = document.getElementById('itemName')?.value.trim();
      const price = parseFloat(document.getElementById('itemPrice')?.value);
      const category = document.getElementById('itemCategory')?.value;
      const description = document.getElementById('itemDescription')?.value;
      const file = document.getElementById('itemFile')?.files[0];
      
      if (!name || !price || isNaN(price) || price <= 0) {
        showNotification('Please enter valid name and price', 'error');
        return;
      }
      
      if (editingItem) {
        await updateMenuItem(editingItem.id, { name, price, category, description }, file);
      } else {
        await addMenuItem({ name, price, category, description }, file);
      }
      closeModal();
    });
  }
  
  // ======================== INITIALIZATION ========================
  function init() {
    render();
  }
  
  init();
</script>
</body>
</html>
