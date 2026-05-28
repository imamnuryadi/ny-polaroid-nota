<img width="3656" height="3656" alt="ny-polaroid-logo" src="https://github.com/user-attachments/assets/895cb82c-bf8f-425a-9109-cf863f567c47" />
const STORE = {
  name: "NY POLAROID",
  address: "Perumahan Griya Putra Mangunjiwan Asri 5 RT 14 RW 01 Demak",
  phone: "0895388957398",
  instagram: "@ny.polaroid",
};

const STORAGE_KEY = "ny-polaroid-orders";

const form = document.querySelector("#orderForm");[README.md](https://github.com/user-attachments/files/28350536/README.md)
[styles.css](https://github.com/user-attachments/files/28350535/styles.css)
[index.html](https://github.com/user-attachments/files/28350534/index.html)
[script.js](https://github.com/user-attachments/files/28350533/script.js)

const invoiceNo = document.querySelector("#invoiceNo");
const orderDate = document.querySelector("#orderDate");
const customerName = document.querySelector("#customerName");
const customerPhone = document.querySelector("#customerPhone");
const customerAddress = document.querySelector("#customerAddress");
const itemsBody = document.querySelector("#itemsBody");
const shippingCost = document.querySelector("#shippingCost");
const discount = document.querySelector("#discount");
const grandTotal = document.querySelector("#grandTotal");
const addItemBtn = document.querySelector("#addItemBtn");
const printBtn = document.querySelector("#printBtn");
const downloadJpegBtn = document.querySelector("#downloadJpegBtn");
const newOrderBtn = document.querySelector("#newOrderBtn");
const ordersList = document.querySelector("#ordersList");
const orderCount = document.querySelector("#orderCount");
const printArea = document.querySelector("#printArea");
const statusMessage = document.querySelector("#statusMessage");

let activeOrderId = null;

const rupiah = new Intl.NumberFormat("id-ID", {
  style: "currency",
  currency: "IDR",
  maximumFractionDigits: 0,
});

function todayValue() {
  const now = new Date();
  const year = now.getFullYear();
  const month = String(now.getMonth() + 1).padStart(2, "0");
  const day = String(now.getDate()).padStart(2, "0");
  return `${year}-${month}-${day}`;
}

function createInvoiceNo() {
  const now = new Date();
  const datePart = todayValue().replaceAll("-", "");
  const timePart = String(now.getHours()).padStart(2, "0") + String(now.getMinutes()).padStart(2, "0") + String(now.getSeconds()).padStart(2, "0");
  return `NY-${datePart}-${timePart}`;
}

function readOrders() {
  return JSON.parse(localStorage.getItem(STORAGE_KEY) || "[]");
}

function writeOrders(orders) {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(orders));
}

function numberValue(value) {
  return Math.max(0, Number(value) || 0);
}

function percentValue(value) {
  return Math.min(100, numberValue(value));
}

function orderDiscountPercent(order) {
  return percentValue(order.discountPercent ?? order.discount ?? 0);
}

function orderDiscountAmount(order) {
  return order.discountAmount ?? (order.subtotal || 0) * (orderDiscountPercent(order) / 100);
}

function orderTotal(order) {
  return Math.max(0, (order.subtotal || 0) + (order.shippingCost || 0) - orderDiscountAmount(order));
}

function addItemRow(item = {}) {
  const row = document.createElement("div");
  row.className = "item-row";
  row.innerHTML = `
    <input class="item-name" type="text" placeholder="Nama barang" value="${escapeAttribute(item.name || "")}" required />
    <input class="item-qty" type="number" min="1" inputmode="numeric" value="${item.qty || 1}" required />
    <input class="item-price" type="number" min="0" inputmode="numeric" value="${item.price || 0}" required />
    <span class="subtotal">Rp0</span>
    <button class="remove-item" type="button" aria-label="Hapus barang">x</button>
  `;

  row.querySelectorAll("input").forEach((input) => {
    input.addEventListener("input", calculateTotals);
  });

  row.querySelector(".remove-item").addEventListener("click", () => {
    if (itemsBody.children.length === 1) {
      row.querySelector(".item-name").value = "";
      row.querySelector(".item-qty").value = 1;
      row.querySelector(".item-price").value = 0;
    } else {
      row.remove();
    }
    calculateTotals();
  });

  itemsBody.appendChild(row);
  calculateTotals();
}

function escapeAttribute(value) {
  return String(value)
    .replaceAll("&", "&amp;")
    .replaceAll('"', "&quot;")
    .replaceAll("<", "&lt;")
    .replaceAll(">", "&gt;");
}

function collectItems() {
  return Array.from(itemsBody.querySelectorAll(".item-row"))
    .map((row) => {
      const name = row.querySelector(".item-name").value.trim();
      const qty = numberValue(row.querySelector(".item-qty").value);
      const price = numberValue(row.querySelector(".item-price").value);
      return {
        name,
        qty,
        price,
        subtotal: qty * price,
      };
    })
    .filter((item) => item.name);
}

function calculateTotals() {
  let itemTotal = 0;
  itemsBody.querySelectorAll(".item-row").forEach((row) => {
    const qty = numberValue(row.querySelector(".item-qty").value);
    const price = numberValue(row.querySelector(".item-price").value);
    const subtotal = qty * price;
    itemTotal += subtotal;
    row.querySelector(".subtotal").textContent = rupiah.format(subtotal);
  });

  const discountPercent = percentValue(discount.value);
  const discountAmount = itemTotal * (discountPercent / 100);
  const total = itemTotal + numberValue(shippingCost.value) - discountAmount;
  grandTotal.textContent = rupiah.format(Math.max(0, total));
}

function setStatus(message) {
  statusMessage.textContent = message;
  if (message) {
    window.setTimeout(() => {
      if (statusMessage.textContent === message) statusMessage.textContent = "";
    }, 3500);
  }
}

function collectOrder() {
  const items = collectItems();
  const subtotal = items.reduce((sum, item) => sum + item.subtotal, 0);
  const shipping = numberValue(shippingCost.value);
  const discountPercent = percentValue(discount.value);
  const discountAmount = subtotal * (discountPercent / 100);

  return {
    id: activeOrderId || crypto.randomUUID(),
    invoiceNo: invoiceNo.value,
    orderDate: orderDate.value,
    customerName: customerName.value.trim(),
    customerPhone: customerPhone.value.trim(),
    customerAddress: customerAddress.value.trim(),
    items,
    shippingCost: shipping,
    discount: discountPercent,
    discountPercent,
    discountAmount,
    subtotal,
    total: Math.max(0, subtotal + shipping - discountAmount),
    updatedAt: new Date().toISOString(),
  };
}

function saveOrder() {
  const order = collectOrder();
  if (!order.items.length) {
    alert("Tambahkan minimal 1 barang terlebih dahulu.");
    return null;
  }

  const orders = readOrders();
  const existingIndex = orders.findIndex((item) => item.id === order.id);
  if (existingIndex >= 0) {
    orders[existingIndex] = order;
  } else {
    orders.unshift(order);
  }
  writeOrders(orders);
  activeOrderId = order.id;
  renderOrders();
  return order;
}

function getReadyOrder() {
  if (!form.reportValidity()) return null;
  const order = collectOrder();
  if (!order.items.length) {
    alert("Tambahkan minimal 1 barang terlebih dahulu.");
    return null;
  }
  return order;
}

function resetForm() {
  activeOrderId = null;
  form.reset();
  setStatus("");
  invoiceNo.value = createInvoiceNo();
  orderDate.value = todayValue();
  shippingCost.value = 0;
  discount.value = 0;
  itemsBody.innerHTML = "";
  addItemRow();
  renderPrintArea(collectOrder());
}

function loadOrder(orderId) {
  const order = readOrders().find((item) => item.id === orderId);
  if (!order) return;

  activeOrderId = order.id;
  invoiceNo.value = order.invoiceNo;
  orderDate.value = order.orderDate;
  customerName.value = order.customerName;
  customerPhone.value = order.customerPhone;
  customerAddress.value = order.customerAddress;
  shippingCost.value = order.shippingCost;
  discount.value = order.discountPercent ?? order.discount ?? 0;
  itemsBody.innerHTML = "";
  order.items.forEach(addItemRow);
  renderPrintArea(order);
  window.scrollTo({ top: 0, behavior: "smooth" });
}

function deleteOrder(orderId) {
  const orders = readOrders().filter((item) => item.id !== orderId);
  writeOrders(orders);
  if (activeOrderId === orderId) resetForm();
  renderOrders();
}

function renderOrders() {
  const orders = readOrders();
  orderCount.textContent = `${orders.length} order`;
  ordersList.innerHTML = "";

  if (!orders.length) {
    ordersList.innerHTML = '<p class="empty-orders">Belum ada order tersimpan.</p>';
    return;
  }

  orders.forEach((order) => {
    const item = document.createElement("article");
    item.className = "order-item";
    item.innerHTML = `
      <strong>${escapeHtml(order.customerName || "Tanpa nama")}</strong>
      <div class="order-meta">
        <span>${escapeHtml(order.invoiceNo)}</span>
        <span>${rupiah.format(orderTotal(order))}</span>
      </div>
      <div class="order-meta">
        <span>${formatDate(order.orderDate)}</span>
        <span>${order.items.length} barang</span>
      </div>
      <div class="order-item-actions">
        <button class="mini-button" type="button" data-action="load">Buka</button>
        <button class="mini-button" type="button" data-action="print">Print</button>
        <button class="mini-button" type="button" data-action="jpeg">JPEG</button>
        <button class="mini-button danger" type="button" data-action="delete">Hapus</button>
      </div>
    `;

    item.querySelector('[data-action="load"]').addEventListener("click", () => loadOrder(order.id));
    item.querySelector('[data-action="print"]').addEventListener("click", () => {
      renderPrintArea(order);
      window.print();
    });
    item.querySelector('[data-action="jpeg"]').addEventListener("click", () => downloadReceiptJpeg(order));
    item.querySelector('[data-action="delete"]').addEventListener("click", () => {
      if (confirm("Hapus order ini?")) deleteOrder(order.id);
    });

    ordersList.appendChild(item);
  });
}

function renderPrintArea(order) {
  const itemsHtml = (order.items || [])
    .map(
      (item, index) => `
        <tr>
          <td>${index + 1}. ${escapeHtml(item.name)}</td>
          <td>${item.qty}</td>
          <td>${rupiah.format(item.price)}</td>
          <td>${rupiah.format(item.subtotal)}</td>
        </tr>
      `
    )
    .join("");

  printArea.innerHTML = `
    <article class="receipt">
      <header class="receipt-header">
        <img src="ny-polaroid-logo.jpg" alt="Logo NY POLAROID" class="receipt-logo" />
        <div>
          <h2>${STORE.name}</h2>
          <p>${STORE.address}</p>
          <div class="receipt-contact">
            <span class="contact-item">${iconSvg("whatsapp")}${STORE.phone}</span>
            <span class="contact-item">${iconSvg("instagram")}${STORE.instagram}</span>
          </div>
        </div>
      </header>

      <section class="receipt-info">
        <p><strong>No. Nota:</strong> ${escapeHtml(order.invoiceNo || "-")}</p>
        <p><strong>Tanggal:</strong> ${formatDate(order.orderDate)}</p>
        <p><strong>Nama:</strong> ${escapeHtml(order.customerName || "-")}</p>
        <p><strong>No. WA:</strong> ${escapeHtml(order.customerPhone || "-")}</p>
        <p><strong>Alamat/Catatan:</strong> ${escapeHtml(order.customerAddress || "-")}</p>
      </section>

      <table class="receipt-table">
        <thead>
          <tr>
            <th>Barang</th>
            <th>Qty</th>
            <th>Harga</th>
            <th>Subtotal</th>
          </tr>
        </thead>
        <tbody>
          ${itemsHtml || '<tr><td colspan="4">Belum ada barang.</td></tr>'}
          <tr class="receipt-total">
            <td colspan="3">Subtotal</td>
            <td>${rupiah.format(order.subtotal || 0)}</td>
          </tr>
          <tr>
            <td colspan="3">Ongkir</td>
            <td>${rupiah.format(order.shippingCost || 0)}</td>
          </tr>
          <tr>
            <td colspan="3">Diskon (${formatPercent(orderDiscountPercent(order))})</td>
            <td>${rupiah.format(orderDiscountAmount(order))}</td>
          </tr>
          <tr class="receipt-total">
            <td colspan="3">Total</td>
            <td>${rupiah.format(orderTotal(order))}</td>
          </tr>
        </tbody>
      </table>

      <footer class="receipt-footer">
        <p>Terima kasih sudah berbelanja di ${STORE.name}.</p>
        <div class="signature">
          <p>Hormat kami,</p>
          <div class="signature-space"></div>
          <p>${STORE.name}</p>
        </div>
      </footer>
    </article>
  `;
}

function escapeHtml(value) {
  return String(value)
    .replaceAll("&", "&amp;")
    .replaceAll("<", "&lt;")
    .replaceAll(">", "&gt;")
    .replaceAll('"', "&quot;")
    .replaceAll("'", "&#039;");
}

function formatDate(value) {
  if (!value) return "-";
  return new Date(`${value}T00:00:00`).toLocaleDateString("id-ID", {
    day: "2-digit",
    month: "long",
    year: "numeric",
  });
}

function formatPercent(value) {
  const percent = percentValue(value);
  return `${percent.toLocaleString("id-ID", { maximumFractionDigits: 2 })}%`;
}

function iconSvg(type) {
  const paths = {
    whatsapp:
      "M19.1 4.9A9.8 9.8 0 0 0 3.7 16.7L2.4 21.5l4.9-1.3A9.8 9.8 0 0 0 21.8 11.9a9.7 9.7 0 0 0-2.7-7Zm-7.2 15a7.9 7.9 0 0 1-4-1.1l-.3-.2-2.9.8.8-2.8-.2-.3A7.9 7.9 0 1 1 11.9 19.9Zm4.3-5.9c-.2-.1-1.4-.7-1.6-.8-.2-.1-.4-.1-.6.1-.2.2-.6.8-.8 1-.1.2-.3.2-.5.1a6.5 6.5 0 0 1-3.2-2.8c-.2-.3 0-.4.1-.6l.4-.5c.1-.2.2-.3.3-.5.1-.2 0-.4 0-.5l-.8-1.9c-.2-.5-.4-.4-.6-.4h-.5c-.2 0-.5.1-.7.3-.2.2-.9.9-.9 2.1s.9 2.4 1 2.6c.1.2 1.7 2.7 4.2 3.8.6.2 1 .4 1.4.5.6.2 1.1.2 1.5.1.5-.1 1.4-.6 1.6-1.1.2-.5.2-1 .1-1.1-.1-.2-.2-.2-.4-.3Z",
    instagram:
      "M7.8 2.5h8.4a5.3 5.3 0 0 1 5.3 5.3v8.4a5.3 5.3 0 0 1-5.3 5.3H7.8a5.3 5.3 0 0 1-5.3-5.3V7.8a5.3 5.3 0 0 1 5.3-5.3Zm8.4 17.2a3.5 3.5 0 0 0 3.5-3.5V7.8a3.5 3.5 0 0 0-3.5-3.5H7.8a3.5 3.5 0 0 0-3.5 3.5v8.4a3.5 3.5 0 0 0 3.5 3.5h8.4ZM12 7.1a4.9 4.9 0 1 1 0 9.8 4.9 4.9 0 0 1 0-9.8Zm0 8a3.1 3.1 0 1 0 0-6.2 3.1 3.1 0 0 0 0 6.2Zm5.2-8.5a1.2 1.2 0 1 1 0 2.4 1.2 1.2 0 0 1 0-2.4Z",
  };
  return `<svg class="contact-icon" viewBox="0 0 24 24" aria-hidden="true"><path d="${paths[type]}"></path></svg>`;
}

async function downloadReceiptJpeg(order) {
  const canvas = document.createElement("canvas");
  const width = 900;
  const rowHeight = 42;
  const baseHeight = 560;
  const height = baseHeight + Math.max(order.items.length, 1) * rowHeight;
  const margin = 48;
  const ctx = canvas.getContext("2d");

  canvas.width = width;
  canvas.height = height;
  ctx.fillStyle = "#ffffff";
  ctx.fillRect(0, 0, width, height);
  ctx.fillStyle = "#111111";
  ctx.strokeStyle = "#111111";
  ctx.lineWidth = 2;
  ctx.font = "18px Arial";

  await drawLogo(ctx, margin, 36, 96, 96);
  ctx.font = "700 30px Arial";
  ctx.fillText(STORE.name, 164, 62);
  ctx.font = "16px Arial";
  wrapText(ctx, STORE.address, 164, 92, 640, 22);
  drawMiniIcon(ctx, "WA", 164, 124);
  ctx.fillText(STORE.phone, 194, 140);
  drawMiniIcon(ctx, "IG", 332, 124);
  ctx.fillText(STORE.instagram, 362, 140);
  line(ctx, margin, 160, width - margin, 160, 3);

  ctx.font = "700 17px Arial";
  ctx.fillText("No. Nota", margin, 196);
  ctx.fillText("Tanggal", 470, 196);
  ctx.fillText("Nama", margin, 230);
  ctx.fillText("No. WA", 470, 230);
  ctx.fillText("Alamat/Catatan", margin, 264);

  ctx.font = "16px Arial";
  ctx.fillText(`: ${order.invoiceNo || "-"}`, 170, 196);
  ctx.fillText(`: ${formatDate(order.orderDate)}`, 570, 196);
  ctx.fillText(`: ${order.customerName || "-"}`, 170, 230);
  ctx.fillText(`: ${order.customerPhone || "-"}`, 570, 230);
  wrapText(ctx, `: ${order.customerAddress || "-"}`, 170, 264, 650, 22);

  const tableTop = 322;
  const col = [margin, 475, 560, 705, width - margin];
  ctx.font = "700 16px Arial";
  ctx.fillStyle = "#124b67";
  ctx.fillRect(margin, tableTop, width - margin * 2, rowHeight);
  ctx.fillStyle = "#ffffff";
  ctx.fillText("Barang", col[0] + 12, tableTop + 27);
  ctx.fillText("Qty", col[1] + 18, tableTop + 27);
  ctx.fillText("Harga", col[2] + 14, tableTop + 27);
  ctx.fillText("Subtotal", col[3] + 14, tableTop + 27);

  ctx.strokeStyle = "#111111";
  ctx.fillStyle = "#111111";
  const rows = order.items.length ? order.items : [{ name: "Belum ada barang", qty: "", price: 0, subtotal: 0 }];
  rows.forEach((item, index) => {
    const y = tableTop + rowHeight * (index + 1);
    ctx.strokeRect(margin, y, width - margin * 2, rowHeight);
    col.slice(1, 4).forEach((x) => line(ctx, x, y, x, y + rowHeight, 1));
    ctx.font = "15px Arial";
    ctx.fillText(`${index + 1}. ${item.name}`, col[0] + 12, y + 27);
    centerText(ctx, String(item.qty), col[1], col[2], y + 27);
    rightText(ctx, rupiah.format(item.price), col[3] - 12, y + 27);
    rightText(ctx, rupiah.format(item.subtotal), col[4] - 12, y + 27);
  });

  const totalTop = tableTop + rowHeight * (rows.length + 1) + 22;
  drawTotalLine(ctx, "Subtotal", order.subtotal || 0, totalTop);
  drawTotalLine(ctx, "Ongkir", order.shippingCost || 0, totalTop + 32);
  drawTotalLine(ctx, `Diskon (${formatPercent(orderDiscountPercent(order))})`, orderDiscountAmount(order), totalTop + 64);
  ctx.fillStyle = "#124b67";
  ctx.fillRect(520, totalTop + 86, 332, 48);
  ctx.fillStyle = "#ffffff";
  ctx.font = "700 20px Arial";
  ctx.fillText("TOTAL", 540, totalTop + 117);
  rightText(ctx, rupiah.format(orderTotal(order)), 830, totalTop + 117);

  ctx.fillStyle = "#111111";
  ctx.font = "16px Arial";
  ctx.fillText(`Terima kasih sudah berbelanja di ${STORE.name}.`, margin, height - 94);
  centerText(ctx, "Hormat kami,", 650, 850, height - 112);
  centerText(ctx, STORE.name, 650, 850, height - 42);

  const link = document.createElement("a");
  link.href = canvas.toDataURL("image/jpeg", 0.92);
  link.download = `${sanitizeFileName(order.invoiceNo || "nota-ny-polaroid")}.jpg`;
  link.click();
  setStatus("Nota JPEG berhasil dibuat.");
}

function drawTotalLine(ctx, label, value, y) {
  ctx.fillStyle = "#111111";
  ctx.font = "16px Arial";
  ctx.fillText(label, 540, y);
  rightText(ctx, rupiah.format(value), 830, y);
}

function drawMiniIcon(ctx, label, x, y) {
  ctx.save();
  ctx.fillStyle = "#124b67";
  ctx.beginPath();
  ctx.arc(x + 10, y + 10, 10, 0, Math.PI * 2);
  ctx.fill();
  ctx.fillStyle = "#ffffff";
  ctx.font = "700 8px Arial";
  centerText(ctx, label, x, x + 20, y + 13);
  ctx.restore();
}

function drawLogo(ctx, x, y, width, height) {
  return new Promise((resolve) => {
    const image = new Image();
    image.onload = () => {
      ctx.drawImage(image, x, y, width, height);
      resolve();
    };
    image.onerror = resolve;
    image.src = "ny-polaroid-logo.jpg";
  });
}

function line(ctx, x1, y1, x2, y2, width = 1) {
  ctx.save();
  ctx.lineWidth = width;
  ctx.beginPath();
  ctx.moveTo(x1, y1);
  ctx.lineTo(x2, y2);
  ctx.stroke();
  ctx.restore();
}

function rightText(ctx, text, x, y) {
  ctx.fillText(text, x - ctx.measureText(text).width, y);
}

function centerText(ctx, text, left, right, y) {
  const x = left + (right - left - ctx.measureText(text).width) / 2;
  ctx.fillText(text, x, y);
}

function wrapText(ctx, text, x, y, maxWidth, lineHeight) {
  const words = String(text).split(" ");
  let lineText = "";
  words.forEach((word, index) => {
    const testLine = lineText ? `${lineText} ${word}` : word;
    if (ctx.measureText(testLine).width > maxWidth && lineText) {
      ctx.fillText(lineText, x, y);
      lineText = word;
      y += lineHeight;
    } else {
      lineText = testLine;
    }
    if (index === words.length - 1) ctx.fillText(lineText, x, y);
  });
}

function sanitizeFileName(value) {
  return String(value).replace(/[\\/:*?"<>|]/g, "-");
}

addItemBtn.addEventListener("click", () => addItemRow());
shippingCost.addEventListener("input", calculateTotals);
discount.addEventListener("input", calculateTotals);

form.addEventListener("submit", (event) => {
  event.preventDefault();
  const order = saveOrder();
  if (order) {
    renderPrintArea(order);
    setStatus("Order berhasil disimpan.");
  }
});

printBtn.addEventListener("click", () => {
  const order = getReadyOrder();
  if (!order) return;
  renderPrintArea(order);
  window.print();
});

downloadJpegBtn.addEventListener("click", () => {
  const order = getReadyOrder();
  if (!order) return;
  downloadReceiptJpeg(order);
});

newOrderBtn.addEventListener("click", resetForm);

resetForm();
renderOrders();

<!doctype html>
<html lang="id">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Nota Order NY POLAROID</title>
    <link rel="stylesheet" href="styles.css" />
  </head>
  <body>
    <main class="app-shell">
      <section class="workspace" aria-label="Form order nota">
        <header class="store-header">
          <img src="ny-polaroid-logo.jpg" alt="Logo NY POLAROID" class="store-logo" />
          <div>
            <p class="eyebrow">Nota Toko</p>
            <h1>NY POLAROID</h1>
            <p class="store-detail">Perumahan Griya Putra Mangunjiwan Asri 5 RT 14 RW 01 Demak</p>
            <div class="contact-list">
              <span class="contact-item">
                <svg class="contact-icon" viewBox="0 0 24 24" aria-hidden="true">
                  <path d="M19.1 4.9A9.8 9.8 0 0 0 3.7 16.7L2.4 21.5l4.9-1.3A9.8 9.8 0 0 0 21.8 11.9a9.7 9.7 0 0 0-2.7-7Zm-7.2 15a7.9 7.9 0 0 1-4-1.1l-.3-.2-2.9.8.8-2.8-.2-.3A7.9 7.9 0 1 1 11.9 19.9Zm4.3-5.9c-.2-.1-1.4-.7-1.6-.8-.2-.1-.4-.1-.6.1-.2.2-.6.8-.8 1-.1.2-.3.2-.5.1a6.5 6.5 0 0 1-3.2-2.8c-.2-.3 0-.4.1-.6l.4-.5c.1-.2.2-.3.3-.5.1-.2 0-.4 0-.5l-.8-1.9c-.2-.5-.4-.4-.6-.4h-.5c-.2 0-.5.1-.7.3-.2.2-.9.9-.9 2.1s.9 2.4 1 2.6c.1.2 1.7 2.7 4.2 3.8.6.2 1 .4 1.4.5.6.2 1.1.2 1.5.1.5-.1 1.4-.6 1.6-1.1.2-.5.2-1 .1-1.1-.1-.2-.2-.2-.4-.3Z" />
                </svg>
                0895388957398
              </span>
              <span class="contact-item">
                <svg class="contact-icon" viewBox="0 0 24 24" aria-hidden="true">
                  <path d="M7.8 2.5h8.4a5.3 5.3 0 0 1 5.3 5.3v8.4a5.3 5.3 0 0 1-5.3 5.3H7.8a5.3 5.3 0 0 1-5.3-5.3V7.8a5.3 5.3 0 0 1 5.3-5.3Zm8.4 17.2a3.5 3.5 0 0 0 3.5-3.5V7.8a3.5 3.5 0 0 0-3.5-3.5H7.8a3.5 3.5 0 0 0-3.5 3.5v8.4a3.5 3.5 0 0 0 3.5 3.5h8.4ZM12 7.1a4.9 4.9 0 1 1 0 9.8 4.9 4.9 0 0 1 0-9.8Zm0 8a3.1 3.1 0 1 0 0-6.2 3.1 3.1 0 0 0 0 6.2Zm5.2-8.5a1.2 1.2 0 1 1 0 2.4 1.2 1.2 0 0 1 0-2.4Z" />
                </svg>
                @ny.polaroid
              </span>
            </div>
          </div>
        </header>

        <form id="orderForm" class="order-form">
          <div class="form-grid">
            <label>
              No. Nota
              <input id="invoiceNo" name="invoiceNo" type="text" readonly />
            </label>
            <label>
              Tanggal
              <input id="orderDate" name="orderDate" type="date" required />
            </label>
            <label>
              Nama Pembeli
              <input id="customerName" name="customerName" type="text" placeholder="Nama pembeli" required />
            </label>
            <label>
              No. WA Pembeli
              <input id="customerPhone" name="customerPhone" type="tel" placeholder="08xxxxxxxxxx" />
            </label>
          </div>

          <label>
            Alamat / Catatan Order
            <textarea id="customerAddress" name="customerAddress" rows="3" placeholder="Alamat pengiriman atau catatan tambahan"></textarea>
          </label>

          <section class="items-section" aria-label="Rincian barang">
            <div class="section-title">
              <h2>Rincian Barang</h2>
              <button class="secondary-button" type="button" id="addItemBtn">Tambah Barang</button>
            </div>

            <div class="items-table">
              <div class="items-head">
                <span>Barang</span>
                <span>Qty</span>
                <span>Harga</span>
                <span>Subtotal</span>
                <span></span>
              </div>
              <div id="itemsBody"></div>
            </div>
          </section>

          <section class="payment-section" aria-label="Ringkasan pembayaran">
            <label>
              Ongkir
              <input id="shippingCost" name="shippingCost" type="number" min="0" inputmode="numeric" value="0" />
            </label>
            <label>
              Diskon (%)
              <input id="discount" name="discount" type="number" min="0" max="100" step="0.01" inputmode="decimal" value="0" />
            </label>
            <div class="grand-total">
              <span>Total</span>
              <strong id="grandTotal">Rp0</strong>
            </div>
          </section>

          <div class="actions">
            <button type="submit" class="primary-button">Simpan Order</button>
            <button type="button" class="secondary-button" id="printBtn">Print Nota</button>
            <button type="button" class="secondary-button" id="downloadJpegBtn">Download JPEG</button>
            <button type="button" class="ghost-button" id="newOrderBtn">Order Baru</button>
          </div>
          <p id="statusMessage" class="status-message" aria-live="polite"></p>
        </form>
      </section>

      <aside class="orders-panel" aria-label="Daftar order masuk">
        <div class="section-title compact">
          <h2>Order Masuk</h2>
          <span id="orderCount">0 order</span>
        </div>
        <div id="ordersList" class="orders-list"></div>
      </aside>
    </main>

    <section id="printArea" class="print-area" aria-label="Nota cetak"></section>

    <script src="script.js"></script>
  </body>
</html>

:root {
  --ink: #17212b;
  --muted: #637083;
  --line: #d9e1e8;
  --paper: #ffffff;
  --wash: #f5f7fa;
  --brand: #124b67;
  --brand-dark: #0d3548;
  --accent: #e8b35b;
  --danger: #b42318;
  --radius: 8px;
}

* {
  box-sizing: border-box;
}

body {
  margin: 0;
  background: var(--wash);
  color: var(--ink);
  font-family: Arial, Helvetica, sans-serif;
  font-size: 15px;
  line-height: 1.45;
}

button,
input,
textarea {
  font: inherit;
}

button {
  border: 0;
  cursor: pointer;
}

.app-shell {
  display: grid;
  grid-template-columns: minmax(0, 1fr) 340px;
  gap: 18px;
  max-width: 1280px;
  margin: 0 auto;
  padding: 22px;
}

.workspace,
.orders-panel {
  background: var(--paper);
  border: 1px solid var(--line);
  border-radius: var(--radius);
}

.workspace {
  padding: 24px;
}

.orders-panel {
  align-self: start;
  padding: 18px;
  position: sticky;
  top: 18px;
}

.store-header {
  display: grid;
  grid-template-columns: 112px minmax(0, 1fr);
  gap: 18px;
  align-items: center;
  padding-bottom: 20px;
  border-bottom: 3px solid var(--brand);
}

.store-logo {
  width: 112px;
  height: 112px;
  object-fit: cover;
  border-radius: var(--radius);
  background: var(--brand);
}

.eyebrow {
  margin: 0 0 3px;
  color: var(--accent);
  font-size: 13px;
  font-weight: 700;
  letter-spacing: 0;
  text-transform: uppercase;
}

h1,
h2,
p {
  margin-top: 0;
}

h1 {
  margin-bottom: 6px;
  color: var(--brand-dark);
  font-size: 34px;
  line-height: 1.05;
}

h2 {
  margin-bottom: 0;
  font-size: 18px;
}

.store-detail {
  margin-bottom: 2px;
  color: var(--muted);
}

.contact-list {
  display: flex;
  flex-wrap: wrap;
  gap: 8px 16px;
  margin-top: 8px;
}

.contact-item {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  color: var(--brand-dark);
  font-weight: 700;
}

.contact-icon {
  width: 20px;
  height: 20px;
  flex: 0 0 auto;
  fill: var(--brand);
}

.order-form {
  display: grid;
  gap: 18px;
  padding-top: 20px;
}

.form-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 14px;
}

label {
  display: grid;
  gap: 7px;
  color: var(--brand-dark);
  font-weight: 700;
}

input,
textarea {
  width: 100%;
  border: 1px solid var(--line);
  border-radius: var(--radius);
  color: var(--ink);
  background: #fff;
  padding: 11px 12px;
  outline: none;
  font-weight: 400;
}

input:focus,
textarea:focus {
  border-color: var(--brand);
  box-shadow: 0 0 0 3px rgba(18, 75, 103, 0.12);
}

textarea {
  resize: vertical;
}

.section-title {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  margin-bottom: 12px;
}

.section-title.compact {
  margin-bottom: 14px;
}

.section-title span {
  color: var(--muted);
  font-size: 13px;
}

.items-table {
  overflow-x: auto;
  border: 1px solid var(--line);
  border-radius: var(--radius);
}

.items-head,
.item-row {
  display: grid;
  grid-template-columns: minmax(190px, 1fr) 86px 132px 132px 44px;
  gap: 8px;
  align-items: center;
  min-width: 680px;
  padding: 10px;
}

.items-head {
  background: #eef3f7;
  color: var(--brand-dark);
  font-size: 13px;
  font-weight: 700;
}

.item-row + .item-row {
  border-top: 1px solid var(--line);
}

.item-row input {
  padding: 9px 10px;
}

.subtotal {
  color: var(--ink);
  font-weight: 700;
  text-align: right;
}

.remove-item {
  width: 34px;
  height: 34px;
  border-radius: var(--radius);
  background: #fdecec;
  color: var(--danger);
  font-size: 20px;
  line-height: 1;
}

.payment-section {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 170px)) minmax(220px, 1fr);
  gap: 14px;
  align-items: end;
}

.grand-total {
  display: flex;
  align-items: center;
  justify-content: space-between;
  min-height: 48px;
  padding: 12px 16px;
  border-radius: var(--radius);
  background: var(--brand);
  color: #fff;
}

.grand-total span {
  font-weight: 700;
}

.grand-total strong {
  font-size: 24px;
}

.actions {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.status-message {
  min-height: 22px;
  margin: -6px 0 0;
  color: var(--brand);
  font-weight: 700;
}

.primary-button,
.secondary-button,
.ghost-button {
  min-height: 42px;
  border-radius: var(--radius);
  padding: 10px 16px;
  font-weight: 700;
}

.primary-button {
  background: var(--brand);
  color: #fff;
}

.secondary-button {
  border: 1px solid var(--brand);
  background: #fff;
  color: var(--brand);
}

.ghost-button {
  border: 1px solid var(--line);
  background: #fff;
  color: var(--muted);
}

.orders-list {
  display: grid;
  gap: 10px;
}

.empty-orders {
  color: var(--muted);
  padding: 22px 0;
  text-align: center;
}

.order-item {
  display: grid;
  gap: 8px;
  border: 1px solid var(--line);
  border-radius: var(--radius);
  padding: 12px;
  background: #fff;
}

.order-item strong {
  color: var(--brand-dark);
}

.order-meta {
  display: flex;
  justify-content: space-between;
  gap: 10px;
  color: var(--muted);
  font-size: 13px;
}

.order-item-actions {
  display: flex;
  gap: 8px;
}

.mini-button {
  flex: 1;
  min-height: 34px;
  border-radius: var(--radius);
  background: #eef3f7;
  color: var(--brand-dark);
  font-weight: 700;
}

.mini-button.danger {
  background: #fdecec;
  color: var(--danger);
}

.print-area {
  display: none;
}

@media (max-width: 920px) {
  .app-shell {
    grid-template-columns: 1fr;
  }

  .orders-panel {
    position: static;
  }
}

@media (max-width: 640px) {
  .app-shell {
    padding: 12px;
  }

  .workspace,
  .orders-panel {
    padding: 16px;
  }

  .store-header,
  .form-grid,
  .payment-section {
    grid-template-columns: 1fr;
  }

  .store-logo {
    width: 92px;
    height: 92px;
  }

  h1 {
    font-size: 28px;
  }

  .actions button {
    width: 100%;
  }
}

@media print {
  body {
    background: #fff;
  }

  .app-shell {
    display: none;
  }

  .print-area {
    display: block;
    color: #111;
    font-size: 12px;
  }

  .receipt {
    width: 100%;
    max-width: 760px;
    margin: 0 auto;
    padding: 0;
  }

  .receipt-header {
    display: grid;
    grid-template-columns: 74px 1fr;
    gap: 12px;
    align-items: center;
    border-bottom: 2px solid #111;
    padding-bottom: 10px;
    margin-bottom: 12px;
  }

  .receipt-logo {
    width: 74px;
    height: 74px;
    object-fit: cover;
  }

  .receipt h2 {
    margin: 0 0 4px;
    font-size: 22px;
  }

  .receipt p {
    margin: 0 0 2px;
  }

  .receipt-contact {
    display: flex;
    gap: 16px;
    align-items: center;
    margin-top: 3px;
  }

  .receipt-contact .contact-icon {
    width: 14px;
    height: 14px;
    fill: #111;
  }

  .receipt-contact .contact-item {
    color: #111;
    font-weight: 700;
  }

  .receipt-info {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 8px 18px;
    margin-bottom: 12px;
  }

  .receipt-table {
    width: 100%;
    border-collapse: collapse;
    margin-bottom: 12px;
  }

  .receipt-table th,
  .receipt-table td {
    border: 1px solid #111;
    padding: 7px;
    text-align: left;
  }

  .receipt-table th:nth-child(2),
  .receipt-table td:nth-child(2) {
    text-align: center;
  }

  .receipt-table th:nth-child(3),
  .receipt-table th:nth-child(4),
  .receipt-table td:nth-child(3),
  .receipt-table td:nth-child(4),
  .receipt-total td {
    text-align: right;
  }

  .receipt-total {
    font-weight: 700;
  }

  .receipt-footer {
    display: grid;
    grid-template-columns: 1fr 180px;
    gap: 20px;
    margin-top: 28px;
  }

  .signature {
    text-align: center;
  }

  .signature-space {
    height: 54px;
  }
}

# NY POLAROID Nota

Website nota sederhana untuk mendata order masuk, menghitung total belanja, print nota, dan download nota dalam format JPEG.

## Cara Publish ke GitHub Pages

1. Buka https://github.com/new
2. Buat repository baru, contoh: `ny-polaroid-nota`
3. Pilih `Public`
4. Upload file berikut ke repository:
   - `index.html`
   - `styles.css`
   - `script.js`
   - `ny-polaroid-logo.jpg`
5. Buka menu `Settings` repository.
6. Masuk ke `Pages`.
7. Pada `Build and deployment`, pilih:
   - Source: `Deploy from a branch`
   - Branch: `main`
   - Folder: `/root`
8. Klik `Save`.
9. Tunggu beberapa menit sampai muncul URL GitHub Pages.

Contoh URL yang bisa dibuka dari HP:

`https://USERNAME.github.io/ny-polaroid-nota/`
