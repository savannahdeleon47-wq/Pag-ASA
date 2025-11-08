<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Hospital Transparency Portal</title>
  <meta name="description" content="Public view of account codes, allocations, and fund usage." />
  <!-- Tailwind (CDN, no build step—works on GitHub Pages) -->
  <script src="https://cdn.tailwindcss.com"></script>
  <style>
    /* Progress bar track and fill (accessible) */
    .track { background: #e7edf3; }
    .fill  { background: linear-gradient(90deg, #3b82f6, #4f46e5); }
    /* Subtle dotted bg like dashboards */
    .bg-dots {
      background-image: radial-gradient(#e5e7eb 1px, transparent 1px);
      background-size: 12px 12px;
    }
  </style>
</head>
<body class="min-h-screen bg-white text-slate-800">

  <!-- Topbar -->
  <header class="sticky top-0 z-30 bg-white/80 backdrop-blur border-b">
    <div class="max-w-6xl mx-auto px-4 py-3 flex items-center gap-4">
      <span class="px-3 py-1 text-sm font-semibold rounded-full bg-slate-900 text-white">Pag-ASA</span>
      <h1 class="font-bold">Hospital Transparency Portal</h1>
      <span class="ml-auto text-xs text-slate-500">Static demo (GitHub Pages)</span>
    </div>
  </header>

  <!-- Search / Controls -->
  <section class="max-w-6xl mx-auto px-4 py-6">
    <div class="grid md:grid-cols-2 gap-3">
      <label class="block">
        <span class="text-sm text-slate-600">Search (Account code, department, head, hospital)</span>
        <input id="q" type="search" placeholder="e.g., ACC-2024-006 or Radiology"
               class="mt-1 w-full rounded-xl border border-slate-300 px-4 py-2 focus:outline-none focus:ring-2 focus:ring-slate-900" />
      </label>
      <div class="flex items-end gap-2">
        <button id="btnClear" class="px-4 py-2 rounded-xl border">Clear</button>
        <button id="btnExport" class="px-4 py-2 rounded-xl bg-slate-900 text-white">Export JSON</button>
      </div>
    </div>
  </section>

  <!-- Results -->
  <main class="max-w-6xl mx-auto px-4 pb-24">
    <div id="list" class="space-y-6"></div>
    <p id="empty" class="hidden text-center text-slate-500 py-20">No results. Try a different search.</p>
  </main>

  <!-- Template (rendered by JS) -->
  <template id="cardTemplate">
    <article class="rounded-2xl border overflow-hidden">
      <div class="bg-dots px-4 sm:px-6 py-3 flex items-center gap-2">
        <span class="text-xs font-medium px-2.5 py-1 rounded-full bg-slate-100 text-slate-700 hospital-pill"></span>
        <span class="ml-auto text-xs px-2 py-1 rounded-full font-semibold status-badge"></span>
      </div>

      <div class="p-4 sm:p-6 space-y-2">
        <div class="flex items-center gap-3">
          <h2 class="text-xl sm:text-2xl font-extrabold code"></h2>
        </div>

        <a class="dept text-indigo-600 hover:underline text-sm font-semibold" href="#" target="_blank" rel="noopener"></a>

        <div class="grid sm:grid-cols-3 gap-6 pt-2">
          <div class="flex items-center gap-2">
            <svg width="18" height="18" viewBox="0 0 24 24" class="opacity-70"><path fill="currentColor" d="M12 12q-1.25 0-2.125-.875T9 9q0-1.25.875-2.125T12 6q1.25 0 2.125.875T15 9q0 1.25-.875 2.125T12 12m0 8q-3.35 0-6-1.3T2 15q.3-1.85 1.425-3.387T6.4 9.1q1.025-.55 2.137-.825T11 8q.55 0 1.1.05t1.075.15q-1.05.85-1.612 2.05T11 12.5q0 1.6.7 2.975T13.6 18.2q-0.8.3-1.65.55T12 20"/></svg>
            <div>
              <div class="text-xs text-slate-500">Account Head</div>
              <div class="font-medium head"></div>
            </div>
          </div>

          <div>
            <div class="text-xs text-slate-500">Allotted Funds</div>
            <div class="font-extrabold allotted"></div>
          </div>

          <div>
            <div class="text-xs text-slate-500">Last Updated</div>
            <div class="font-medium updated"></div>
          </div>
        </div>

        <!-- Progress -->
        <div class="pt-6">
          <div class="flex items-center justify-between text-xs text-slate-600">
            <span>Fund Usage</span>
            <span class="percent font-semibold"></span>
          </div>
          <div class="mt-2 h-3 w-full rounded-full track overflow-hidden" role="progressbar" aria-valuemin="0" aria-valuemax="100">
            <div class="h-full fill rounded-full" style="width:0%"></div>
          </div>
          <div class="mt-2 flex items-center justify-between text-sm">
            <span>Used: <strong class="used"></strong></span>
            <span class="text-slate-500">Remaining: <strong class="remain"></strong></span>
          </div>
        </div>
      </div>
    </article>
  </template>

  <script>
    // ---- Demo data (add/modify as needed) ----
    const DATA = [
      {
        hospital: "Saelwyn Community Medical Center",
        status: "Active",
        accountCode: "ACC-2024-006",
        department: "Radiology",
        departmentUrl: "#",
        head: "Dr. Robert Johnson",
        allotted: 2800000,
        used: 2200000,
        updated: "2024-01-10"
      },
      {
        hospital: "Aurelia Medical Center",
        status: "Active",
        accountCode: "ACC-2024-010",
        department: "Pharmacy",
        departmentUrl: "#",
        head: "Dr. L. Santos",
        allotted: 1500000,
        used: 640000,
        updated: "2024-02-02"
      },
      {
        hospital: "Saphira Springs General Hospital",
        status: "On Hold",
        accountCode: "ACC-2024-003",
        department: "Surgery",
        departmentUrl: "#",
        head: "Dr. M. Rivera",
        allotted: 3200000,
        used: 3200000,
        updated: "2024-03-15"
      }
    ];

    // ---- Utilities ----
    const $ = (sel, el = document) => el.querySelector(sel);
    const fmt = new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 });
    const byQuery = (row, q) => {
      const hay = [
        row.hospital, row.status, row.accountCode, row.department,
        row.head, row.updated
      ].join(" ").toLowerCase();
      return hay.includes(q.toLowerCase());
    };

    // ---- Render ----
    const list = $("#list");
    const empty = $("#empty");
    const tpl = $("#cardTemplate");

    function statusStyles(s) {
      if (s.toLowerCase() === "active") return "bg-emerald-100 text-emerald-700";
      if (s.toLowerCase() === "on hold") return "bg-amber-100 text-amber-800";
      return "bg-slate-100 text-slate-700";
    }

    function render(rows) {
      list.innerHTML = "";
      if (!rows.length) {
        empty.classList.remove("hidden");
        return;
      }
      empty.classList.add("hidden");

      rows.forEach(r => {
        const node = tpl.content.cloneNode(true);
        $(".hospital-pill", node).textContent = r.hospital;
        const status = $(".status-badge", node);
        status.textContent = r.status;
        status.className = "ml-auto text-xs px-2 py-1 rounded-full font-semibold status-badge " + statusStyles(r.status);

        $(".code", node).textContent = r.accountCode;
        const dept = $(".dept", node);
        dept.textContent = r.department;
        dept.href = r.departmentUrl || "#";

        $(".head", node).textContent = r.head;
        $(".allotted", node).textContent = fmt.format(r.allotted);
        $(".updated", node).textContent = r.updated;

        const percent = r.allotted ? (r.used / r.allotted) * 100 : 0;
        $(".percent", node).textContent = percent.toFixed(1) + "%";
        $(".used", node).textContent = fmt.format(r.used);
        $(".remain", node).textContent = fmt.format(Math.max(r.allotted - r.used, 0));
        $(".track .fill", node).style.width = Math.min(percent, 100) + "%";
        $(".track", node).setAttribute("aria-valuenow", Math.round(Math.min(percent, 100)));

        list.appendChild(node);
      });
    }

    // ---- Search / Actions ----
    const input = $("#q");
    input.addEventListener("input", () => {
      const q = input.value.trim();
      render(q ? DATA.filter(r => byQuery(r, q)) : DATA);
    });

    $("#btnClear").addEventListener("click", () => {
      input.value = "";
      render(DATA);
      input.focus();
    });

    $("#btnExport").addEventListener("click", () => {
      const blob = new Blob([JSON.stringify(DATA, null, 2)], { type: "application/json" });
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url; a.download = "accounts.json"; a.click();
      URL.revokeObjectURL(url);
    });

    // Initial render
    render(DATA);
  </script>

  <!-- Footer -->
  <footer class="border-t">
    <div class="max-w-6xl mx-auto px-4 py-10 text-sm text-slate-500">
      © <span id="y"></span> Pag-ASA Transparency Demo · Static front-end. Replace demo data with real API or CSV.
    </div>
  </footer>
  <script>document.getElementById("y").textContent = new Date().getFullYear()</script>
</body>
</html>
