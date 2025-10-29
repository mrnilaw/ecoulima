const STORAGE_KEY = "unibudget_ulima_pro_v1";
const BUDGET_KEY = "unibudget_ulima_budget_v1";
const NAME_KEY = "unibudget_ulima_name_v1";
const DEFAULT_RATE = 0.27;

let txs = [];
let chart = null;
let budget = Number(localStorage.getItem(BUDGET_KEY)) || 0;

const el = {
  txList: document.getElementById("txList"),
  totalIncome: document.getElementById("totalIncome"),
  totalExpense: document.getElementById("totalExpense"),
  balance: document.getElementById("balance"),
  txCount: document.getElementById("txCount"),
  pieCanvas: document.getElementById("pieChart"),
  chartLegend: document.getElementById("chartLegend"),
  budgetFill: document.getElementById("budgetFill"),
  budgetMeta: document.getElementById("budgetMeta"),
  toast: document.getElementById("toast")
};

document.addEventListener("DOMContentLoaded", () => {
  document.getElementById("date").value = today();
  document.getElementById("from").value = monthStart();
  document.getElementById("to").value = today();
  document.getElementById("studentName").value = localStorage.getItem(NAME_KEY) || "";

  load();

  document.getElementById("txForm").addEventListener("submit", onSubmit);
  document.getElementById("btnClear").addEventListener("click", () => {
    document.getElementById("txForm").reset();
    document.getElementById("date").value = today();
  });
  document.getElementById("btnExport").addEventListener("click", exportCSV);
  document.getElementById("search").addEventListener("input", render);
  document.getElementById("applyFilters").addEventListener("click", render);
  document.getElementById("setBudget").addEventListener("click", setBudgetFromInput);
  document.getElementById("budgetInput").value = budget || "";
  document.getElementById("convBtn").addEventListener("click", handleConvert);
  document.getElementById("resetData").addEventListener("click", resetAll);
  document.getElementById("suggestBtn").addEventListener("click", showSuggestions);
  document.getElementById("downloadCSV").addEventListener("click", exportCSV);
  document.getElementById("studentName").addEventListener("change", (e) => {
    localStorage.setItem(NAME_KEY, e.target.value || "");
  });
  document.getElementById("themeToggle").addEventListener("click", toggleTheme);

  render();
  renderChart();
  updateBudgetUI();
});

function load() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    txs = raw ? JSON.parse(raw) : [];
  } catch (e) {
    console.error(e);
    txs = [];
  }
}
function save() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(txs));
}

function onSubmit(e) {
  e.preventDefault();
  const id = document.getElementById("txForm").dataset.editId || uid();
  const data = {
    id,
    date: document.getElementById("date").value,
    category: document.getElementById("category").value,
    type: document.getElementById("type").value,
    amount: parseFloat(document.getElementById("amount").value),
    note: document.getElementById("note").value || ""
  };
  if (!data.date || !data.amount || data.amount <= 0) {
    showToast("Fecha y monto válidos son obligatorios");
    return;
  }

  const editIndex = txs.findIndex(t => t.id === id);
  if (editIndex >= 0) {
    txs[editIndex] = data;
    showToast("Registro actualizado");
  } else {
    txs.unshift(data);
    showToast("Registro agregado");
  }
  save();
  document.getElementById("txForm").reset();
  document.getElementById("date").value = today();
  delete document.getElementById("txForm").dataset.editId;

  render();
  renderChart();
  updateBudgetUI();
}

function render() {
  const query = (document.getElementById("search").value || "").toLowerCase().trim();
  const from = document.getElementById("from").value;
  const to = document.getElementById("to").value;
  const view = document.getElementById("viewFilter").value;

  const filtered = txs.filter(t => {
    if (view !== "all" && t.type !== view) return false;
    if (query && !((t.note || "").toLowerCase().includes(query) || (t.category || "").toLowerCase().includes(query))) return false;
    if (from && t.date < from) return false;
    if (to && t.date > to) return false;
    return true;
  });

  el.txList.innerHTML = filtered.length
    ? ""
    : "<li class='tx-item'><div class='tx-left'><div class='tx-meta'>No hay registros</div></div></li>";

  filtered.forEach(t => {
    const li = document.createElement("li");
    li.className = "tx-item";
    const left = document.createElement("div");
    left.className = "tx-left";
    const icon = document.createElement("div");
    icon.className = "icon-circle";
    icon.textContent = initials(t.category);
    const meta = document.createElement("div");
    meta.innerHTML = `<div style="font-weight:700">${t.note || t.category}</div><div class="tx-meta">${t.category} • ${t.date}</div>`;
    left.appendChild(icon);
    left.appendChild(meta);

    const right = document.createElement("div");
    right.style.textAlign = "right";
    const amount = document.createElement("div");
    amount.className = "tx-amount " + (t.type === "ingreso" ? "income" : "expense");
    amount.textContent = (t.type === "gasto" ? "-S/ " : "+S/ ") + Number(t.amount).toFixed(2);

    const actions = document.createElement("div");
    actions.className = "tx-actions";
    const editBtn = document.createElement("button");
    editBtn.className = "btn-ghost";
    editBtn.textContent = "Editar";
    editBtn.onclick = () => openEdit(t.id);
    const delBtn = document.createElement("button");
    delBtn.className = "btn-ghost";
    delBtn.style.color = "var(--danger)";
    delBtn.textContent =
