<script>
  import { onMount } from 'svelte';

  export let csvUrl = 'https://docs.google.com/spreadsheets/d/1Y7xvYKTIpy7xsQOtJWpPfEhQ8gvg3dLjMY_5PzoeglQ/export?format=csv';

  // ─── Constants ──────────────────────────────────────────────────────────
  const NO_MONTHS = [
    'Januar','Februar','Mars','April','Mai','Juni',
    'Juli','August','September','Oktober','November','Desember'
  ];
  const NO_DAYS_SHORT = ['Man','Tir','Ons','Tor','Fre','Lør','Søn'];
  const NO_DAYS_FULL  = ['Mandag','Tirsdag','Onsdag','Torsdag','Fredag','Lørdag','Søndag'];
  const MONTH_MAP = {
    januar:0, februar:1, mars:2, april:3, mai:4, juni:5,
    juli:6, august:7, september:8, oktober:9, november:10, desember:11
  };

  // Season boundaries: Mai 2026 → April 2027
  const MIN_YEAR = 2026, MIN_MONTH = 4;
  const MAX_YEAR = 2027, MAX_MONTH = 3;

  // ─── State ──────────────────────────────────────────────────────────────
  let allEvents     = [];
  let trainerCounts = { herman: 0, syver: 0 };
  let loading       = true;
  let error         = null;
  let selectedEvent = null;
  let currentYear   = 2026;
  let currentMonth  = 4;

  let showJunior = true;
  let showHL     = true;
  let showFelles = true;

  // ─── Admin panel ────────────────────────────────────────────────────────
  let showTrainerFilter = false;
  let filterHerman = false;
  let filterSyver  = false;

  // ─── Group styling ──────────────────────────────────────────────────────
  const GROUP = {
    jr:     { label: 'JR',     chip: 'bg-teal-100 text-teal-700 border-l-2 border-teal-500',   panel: 'bg-teal-600', text: 'text-teal-600' },
    hl:     { label: 'HL',     chip: 'bg-amber-100 text-amber-700 border-l-2 border-amber-500', panel: 'bg-amber-500', text: 'text-amber-500' },
    felles: { label: 'Felles', chip: 'bg-lime-100 text-lime-700 border-l-2 border-lime-500',   panel: 'bg-lime-600', text: 'text-lime-500' },
  };

  // ─── Location → Google Maps embed URLs ─────────────────────────────────
  const LOCATION_MAPS = {
    'Gullhella stasjon':
      'https://www.google.com/maps/embed?pb=!1m18!1m12!1m3!1d2006.540609475089!2d10.433799977792807!3d59.80693827484918!2m3!1f0!2f0!3f0!3m2!1i1024!2i768!4f13.1!3m3!1m2!1s0x464115853d93359d%3A0xba0ee072a98b4fce!2sGullhella%20stasjon!5e0!3m2!1sno!2sno!4v1773848535311!5m2!1sno!2sno',
    'Konnerud':
      'https://www.google.com/maps/embed?pb=!1m18!1m12!1m3!1d6380.5164796917215!2d10.13755929786328!3d59.71519961863389!2m3!1f0!2f0!3f0!3m2!1i1024!2i768!4f13.1!3m3!1m2!1s0x464124964c14414b%3A0x2208a6c50c12144a!2sSletta%20Klubbhus%20Konnerud%20IL!5e0!3m2!1sno!2sno!4v1773848873224!5m2!1sno!2sno',
    'Føyka':
      'https://www.google.com/maps/embed?pb=!1m18!1m12!1m3!1d4009.7083659108257!2d10.423134497925888!3d59.83495341004992!2m3!1f0!2f0!3f0!3m2!1i1024!2i768!4f13.1!3m3!1m2!1s0x464115b7c492cfbf%3A0x97254dd64c49403a!2sF%C3%B8yka%20-%20Asker%20Stadion%2Fskatepark!5e0!3m2!1sno!2sno!4v1773849152408!5m2!1sno!2sno',
    'Lierbakkene':
      'https://www.google.com/maps/embed?pb=!1m18!1m12!1m3!1d2008.938915273757!2d10.280324277790923!3d59.76708357482926!2m3!1f0!2f0!3f0!3m2!1i1024!2i768!4f13.1!3m3!1m2!1s0x46413d00e7e16179%3A0x7b47620dda4719e6!2sLierkroa!5e0!3m2!1sno!2sno!4v1773849239768!5m2!1sno!2sno',
    'Heggedal stasjon':
      'https://www.google.com/maps/embed?pb=!1m18!1m12!1m3!1d2007.7534204347933!2d10.428579077791847!3d59.78678597483913!2m3!1f0!2f0!3f0!3m2!1i1024!2i768!4f13.1!3m3!1m2!1s0x46413f357cc5cc6f%3A0x9dab4bcc83ad0ff!2sPendlerparkering%20Heggedal!5e0!3m2!1sno!2sno!4v1773867581367!5m2!1sno!2sno',
    'Semsvann':
      'https://www.google.com/maps/embed?pb=!1m18!1m12!1m3!1d2003.6109370185513!2d10.422611277795088!3d59.85560127487346!2m3!1f0!2f0!3f0!3m2!1i1024!2i768!4f13.1!3m3!1m2!1s0x46411430b9719515%3A0xefd6e75014e41e1f!2sSemsvannet%20utfartsparkering!5e0!3m2!1sno!2sno!4v1773868493384!5m2!1sno!2sno',
    'Holmenkollen':
      'https://www.google.com/maps/embed?pb=!1m18!1m12!1m3!1d3994.0892095627887!2d10.668425977800132!3d59.9645835749282!2m3!1f0!2f0!3f0!3m2!1i1024!2i768!4f13.1!3m3!1m2!1s0x46416d800523872d%3A0x2f4a7b569997bbf3!2sHolmenkollen%20Nasjonalanlegg!5e0!3m2!1sno!2sno!4v1773868556846!5m2!1sno!2sno',
    'Mosetertoppen':
      'https://www.google.com/maps/embed?pb=!1m18!1m12!1m3!1d4057.3336538030067!2d10.488714796576295!3d61.24709502451953!2m3!1f0!2f0!3f0!3m2!1i1024!2i768!4f13.1!3m3!1m2!1s0x466a8a8c6b8cf8b1%3A0x225fcf4f33517337!2sMosetertoppen%20storhytte!5e0!3m2!1sno!2sno!4v1773871616415!5m2!1sno!2sno'
      // Legg til flere steder her:
    // 'Skistadion Konnerud': 'https://www.google.com/maps/embed?pb=...',
  };

  function getMapUrl(location) {
    if (!location) return null;
    // Eksakt treff (case-insensitive)
    const key = Object.keys(LOCATION_MAPS).find(
      k => k.toLowerCase() === location.trim().toLowerCase()
    );
    return key ? LOCATION_MAPS[key] : null;
  }

  // ─── CSV parsing ────────────────────────────────────────────────────────
  function parseCSVLine(line) {
    const cols = [];
    let inQ = false, cur = '';
    for (let i = 0; i < line.length; i++) {
      const c = line[i];
      if (c === '"' && !inQ)         { inQ = true; }
      else if (c === '"' && inQ)     {
        if (line[i+1] === '"') { cur += '"'; i++; }
        else inQ = false;
      } else if (c === ',' && !inQ)  { cols.push(cur); cur = ''; }
      else                           { cur += c; }
    }
    cols.push(cur);
    return cols;
  }

  function parseCSV(text) {
    const lines = text.replace(/\r\n/g,'\n').replace(/\r/g,'\n').split('\n');
    const rows = [];
    for (let i = 1; i < lines.length; i++) {
      const l = lines[i].trim();
      if (l) rows.push(parseCSVLine(l));
    }
    return rows;
  }

  // ─── Date helper ────────────────────────────────────────────────────────
  function parseNorDate(dateStr) {
    const m = dateStr.trim().match(/^(\d+)\.\s+(\S+)$/);
    if (!m) return null;
    const day   = parseInt(m[1]);
    const month = MONTH_MAP[m[2].toLowerCase()];
    if (month === undefined) return null;
    const year = month >= 4 ? 2026 : 2027;
    return new Date(year, month, day);
  }

  // ─── HL/JR text splitter ────────────────────────────────────────────────
  function splitHLJR(text) {
    if (!text) return null;
    const hl1 = text.match(/HL:\s*(.*?)(?=\s*JR:|$)/s);
    const jr1 = text.match(/JR:\s*(.*?)(?=\s*HL:|$)/s);
    if (hl1 || jr1) {
      return { hl: hl1 ? hl1[1].trim() : null, jr: jr1 ? jr1[1].trim() : null };
    }
    return null;
  }

  function splitPipe(text) {
    if (!text || !text.includes('|')) return null;
    const parts = text.split('|').map(p => p.trim());
    let hl = null, jr = null;
    for (const part of parts) {
      if (/^HL\b/i.test(part))      hl = part.replace(/^HL:?\s*/i,'').trim();
      else if (/^JR\b/i.test(part)) jr = part.replace(/^JR:?\s*/i,'').trim();
    }
    return (hl || jr) ? { hl, jr } : null;
  }

  function groupFromName(hva) {
    if (/\bJR\b/.test(hva)) return 'jr';
    if (/\bHL\b/.test(hva)) return 'hl';
    return 'felles';
  }

  function stripGroupSuffix(title) {
    return title.replace(/\s+(HL|JR)\s*$/,'').replace(/^(HL|JR):\s*/,'').trim();
  }

  // ─── Trainer counter (counts all CSV rows, including hva='x') ─────────────
  function countTrainers(rows) {
    let herman = 0, syver = 0;
    for (const row of rows) {
      const [,, dato,, , , trener] = row;
      if (!dato?.trim()) continue;
      const t = (trener || '').toLowerCase();
      // For split trainer fields like "HL: Herman JR: Syver", count each person once per row
      const splitT = splitHLJR(t);
      if (splitT) {
        if ((splitT.hl || '').includes('herman') || (splitT.jr || '').includes('herman')) herman++;
        if ((splitT.hl || '').includes('syver')  || (splitT.jr || '').includes('syver'))  syver++;
      } else {
        if (t.includes('herman')) herman++;
        if (t.includes('syver'))  syver++;
      }
    }
    return { herman, syver };
  }

  // ─── Main event processor ───────────────────────────────────────────────
  // CSV columns: Uke, Dag, Dato, Hva, Beskrivelse, Fokus, Trener, HL/JR Felles?, Sted
  function processRows(rows) {
    const events = [];
    let uid = 0;

    for (const row of rows) {
      const [,, dato, hva, beskrivelse, fokus, trener, felles, sted] = row;
      if (!dato?.trim() || !hva?.trim()) continue;

      const hvaT  = hva.trim();
      if (hvaT === 'x') continue;

      const date = parseNorDate(dato.trim());
      if (!date) continue;

      const fVal  = (felles      || '').trim();
      const beskT = (beskrivelse || '').trim();
      const fokT  = (fokus       || '').trim();
      const trenT = (trener      || '').trim();
      const stedT = (sted        || '').trim();

      const push = (group, title, description, focus, trainer, location) =>
        events.push({ id: uid++, date, dateStr: dato.trim(), group,
          title: title || hvaT, description, focus, trainer, location });

      if (fVal === '✅') {
        push('felles', hvaT, beskT, fokT, trenT, stedT);

      } else if (fVal === '📍') {
        const splitH = splitHLJR(hvaT) || splitPipe(hvaT);
        if (splitH) {
          const splitB = splitHLJR(beskT) || {};
          const splitF = splitHLJR(fokT)  || {};
          const splitT = splitHLJR(trenT) || {};
          if (splitH.hl && splitH.hl !== 'x')
            push('hl', splitH.hl, splitB.hl || beskT, splitF.hl || fokT, splitT.hl || trenT, stedT);
          if (splitH.jr && splitH.jr !== 'x')
            push('jr', splitH.jr, splitB.jr || beskT, splitF.jr || fokT, splitT.jr || trenT, stedT);
        } else {
          push('felles', hvaT, beskT, fokT, trenT, stedT);
        }

      } else if (fVal === '❌') {
        const splitH = splitHLJR(hvaT);
        if (splitH) {
          const splitB = splitHLJR(beskT) || {};
          const splitF = splitHLJR(fokT)  || {};
          const splitT = splitHLJR(trenT) || {};
          if (splitH.hl) push('hl', splitH.hl, splitB.hl || beskT, splitF.hl || fokT, splitT.hl || trenT, stedT);
          if (splitH.jr) push('jr', splitH.jr, splitB.jr || beskT, splitF.jr || fokT, splitT.jr || trenT, stedT);
        } else {
          const grp   = groupFromName(hvaT);
          const title = stripGroupSuffix(hvaT) || hvaT;
          push(grp, title, beskT, fokT, trenT, stedT);
        }

      } else {
        const grp   = groupFromName(hvaT);
        const title = stripGroupSuffix(hvaT) || hvaT;
        push(grp, title, beskT, fokT, trenT, stedT);
      }
    }
    return events;
  }

  // ─── Calendar helpers ───────────────────────────────────────────────────
  function buildCalendarDays(year, month) {
    const first = new Date(year, month, 1);
    const last  = new Date(year, month + 1, 0);
    let dow = first.getDay() - 1;
    if (dow < 0) dow = 6;

    const days = [];
    for (let i = dow; i > 0; i--)
      days.push({ date: new Date(year, month, 1 - i), cur: false });
    for (let d = 1; d <= last.getDate(); d++)
      days.push({ date: new Date(year, month, d), cur: true });
    let n = 1;
    while (days.length < 42)
      days.push({ date: new Date(year, month + 1, n++), cur: false });
    return days;
  }

  function sameDay(a, b) {
    return a.getFullYear() === b.getFullYear() &&
           a.getMonth()    === b.getMonth()    &&
           a.getDate()     === b.getDate();
  }

  function isToday(date) { return sameDay(date, new Date()); }

  // ─── Reactive derived state ─────────────────────────────────────────────
  // Én reaktiv array som kombinerer kalenderdager + filtrerte events.
  // Når showJunior/showHL/showFelles eller allEvents endres, rekalkeres hele
  // arrayen og Svelte re-renderer {#each calDaysWithEvents} automatisk.
  $: calDaysWithEvents = buildCalendarDays(currentYear, currentMonth).map(day => {
    const key = `${day.date.getFullYear()}-${day.date.getMonth()}-${day.date.getDate()}`;
    const anyTrainerFilter = filterHerman || filterSyver;
    const events = allEvents.filter(e => {
      if (`${e.date.getFullYear()}-${e.date.getMonth()}-${e.date.getDate()}` !== key) return false;
      if (e.group === 'jr'     && !showJunior) return false;
      if (e.group === 'hl'     && !showHL)     return false;
      if (e.group === 'felles' && !showFelles) return false;
      if (anyTrainerFilter) {
        const t = (e.trainer || '').toLowerCase();
        const matchHerman = filterHerman && t.includes('herman');
        const matchSyver  = filterSyver  && t.includes('syver');
        if (!matchHerman && !matchSyver) return false;
      }
      return true;
    });
    return { ...day, events };
  });

  // ─── Trener-statistikk ───────────────────────────────────────────────────
  $: countHerman = trainerCounts.herman;
  $: countSyver  = trainerCounts.syver;

  // ─── Mobile list: only days belonging to the current month ──────────────
  $: mobileDays = calDaysWithEvents.filter(d => d.cur);

  // ─── Reactive modal helpers ─────────────────────────────────────────────
  $: mapUrl  = selectedEvent ? getMapUrl(selectedEvent.location) : null;
  $: hasBoth = !!(selectedEvent?.focus && mapUrl);

  // ─── Season-bounded navigation ──────────────────────────────────────────
  $: canPrev = !(currentYear === MIN_YEAR && currentMonth === MIN_MONTH);
  $: canNext = !(currentYear === MAX_YEAR && currentMonth === MAX_MONTH);

  function prevMonth() {
    if (!canPrev) return;
    if (currentMonth === 0) { currentYear--; currentMonth = 11; }
    else currentMonth--;
  }
  function nextMonth() {
    if (!canNext) return;
    if (currentMonth === 11) { currentYear++; currentMonth = 0; }
    else currentMonth++;
  }
  function goToToday() {
    const t = new Date();
    currentYear  = t.getFullYear();
    currentMonth = t.getMonth();
  }

  // ─── Mount ──────────────────────────────────────────────────────────────
  onMount(async () => {
    try {
      const res = await fetch(csvUrl);
      if (!res.ok) throw new Error(`HTTP ${res.status}: ${res.statusText}`);
      const raw = await res.text();
      const parsedRows = parseCSV(raw);
      allEvents     = processRows(parsedRows);
      trainerCounts = countTrainers(parsedRows);

      const today = new Date();
      const hasThisMonth = allEvents.some(e =>
        e.date.getFullYear() === today.getFullYear() &&
        e.date.getMonth()    === today.getMonth()
      );
      if (hasThisMonth) {
        currentYear  = today.getFullYear();
        currentMonth = today.getMonth();
      } else if (allEvents.length > 0) {
        const first = allEvents.reduce((a, b) => a.date < b.date ? a : b);
        currentYear  = first.date.getFullYear();
        currentMonth = first.date.getMonth();
      }
    } catch (e) {
      error = `Kunne ikke laste treningsplan: ${e.message}`;
    } finally {
      loading = false;
    }
  });
</script>

<!-- ═══════════════════════════════════════════════════════════════════════ -->
<!--  TEMPLATE                                                               -->
<!-- ═══════════════════════════════════════════════════════════════════════ -->

{#if loading}
  <div class="flex items-center justify-center min-h-screen bg-gray-50">
    <div class="text-center">
      <div class="w-10 h-10 border-4 border-teal-600 border-t-transparent rounded-full animate-spin mx-auto mb-3"></div>
      <p class="text-sm text-gray-500 font-medium tracking-wide uppercase">Laster kalender…</p>
    </div>
  </div>

{:else if error}
  <div class="flex items-center justify-center min-h-screen bg-gray-50">
    <div class="text-center max-w-md px-6">
      <div class="w-12 h-12 rounded-full bg-red-100 flex items-center justify-center mx-auto mb-4">
        <svg class="w-6 h-6 text-red-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01M10.29 3.86L1.82 18a2 2 0 001.71 3h16.94a2 2 0 001.71-3L13.71 3.86a2 2 0 00-3.42 0z"/>
        </svg>
      </div>
      <h2 class="text-lg font-black text-gray-900 mb-2">Kunne ikke laste treningsplan</h2>
      <p class="text-sm text-gray-500 mb-4">{error}</p>
      <p class="text-xs text-gray-400">Sjekk at Google Sheets-arket er delt offentlig og at URL-en er riktig.</p>
      <button
        on:click={() => location.reload()}
        class="mt-5 px-5 py-2 bg-teal-600 text-white text-sm font-semibold rounded-lg hover:bg-teal-700 transition"
      >Prøv igjen</button>
    </div>
  </div>

{:else}

<div class="min-h-screen bg-gray-100" style="font-family: 'Segoe UI', system-ui, sans-serif;">

  <!-- ── Top bar ──────────────────────────────────────────────────────── -->
  <header class="bg-white border-b border-gray-200 sticky top-0 z-40 shadow-sm">
    <div class="max-w-screen-xl mx-auto px-4 py-3 flex items-center justify-between">
      <span class="text-base font-black tracking-tight text-gray-900 uppercase">Asker S<button
        on:click={() => showTrainerFilter = !showTrainerFilter}
        class="focus:outline-none"
        tabindex="-1"
        aria-label="Vis trenerfilter"
      >K</button>iklubb Langrenn</span>
    </div>
  </header>

  <main class="max-w-screen-xl mx-auto px-4 py-8">

    <!-- ── Month header ─────────────────────────────────────────────── -->
    <div class="flex flex-col sm:flex-row sm:items-end justify-between gap-4 mb-5">
      <div>
        <p class="text-xs font-bold uppercase tracking-widest text-teal-600 mb-1">Sesongoversikt</p>
        <h1 class="text-5xl font-black uppercase text-gray-900 leading-none">
          {NO_MONTHS[currentMonth]} {currentYear}
        </h1>
        <p class="text-sm text-gray-400 mt-2">Klikk på en økt for å se mer informasjon</p>
      </div>

      <div class="flex items-center gap-2">
        <div class="flex rounded-lg border border-gray-300 bg-white overflow-hidden">
          <button
            on:click={prevMonth}
            disabled={!canPrev}
            class="px-3 py-2 text-sm font-bold border-r border-gray-300 transition
              {canPrev ? 'text-gray-600 hover:bg-gray-100' : 'text-gray-300 cursor-not-allowed'}"
            aria-label="Forrige måned"
          >‹</button>
          <button
            on:click={nextMonth}
            disabled={!canNext}
            class="px-3 py-2 text-sm font-bold transition
              {canNext ? 'text-gray-600 hover:bg-gray-100' : 'text-gray-300 cursor-not-allowed'}"
            aria-label="Neste måned"
          >›</button>
        </div>
      </div>
    </div>

    <!-- ── Filters ──────────────────────────────────────────────────── -->
    <div class="bg-white rounded-xl border border-gray-200 px-5 py-4 mb-5 shadow-sm">
      <p class="text-xs font-bold text-gray-400 uppercase tracking-widest mb-3">Filtrer grupper</p>
      <div class="flex flex-wrap gap-5">

        <label class="flex items-center gap-2 cursor-pointer select-none">
          <input type="checkbox" bind:checked={showJunior} class="w-4 h-4 rounded accent-teal-500" />
          <span class="text-sm font-semibold text-teal-700">JR (Junior)</span>
        </label>

        <label class="flex items-center gap-2 cursor-pointer select-none">
          <input type="checkbox" bind:checked={showHL} class="w-4 h-4 rounded accent-amber-500" />
          <span class="text-sm font-semibold text-amber-700">HL (15-16år)</span>
        </label>

        <label class="flex items-center gap-2 cursor-pointer select-none">
          <input type="checkbox" bind:checked={showFelles} class="w-4 h-4 rounded accent-lime-500" />
          <span class="text-sm font-semibold text-lime-700">Felles (HL + JR)</span>
        </label>

      </div>

      <!-- Trener-filter -->
      {#if showTrainerFilter}
      <div class="mt-3 pt-3 border-t border-gray-100">
        <p class="text-xs font-bold text-gray-400 uppercase tracking-widest mb-3">Filtrer trener</p>
        <div class="flex flex-wrap gap-5">

          <label class="flex items-center gap-2 cursor-pointer select-none">
            <input type="checkbox" bind:checked={filterHerman} class="w-4 h-4 rounded accent-gray-500" />
            <span class="text-sm font-semibold text-gray-700">Herman</span>
            <span class="text-xs font-bold bg-gray-100 text-gray-600 rounded-full px-2 py-0.5">{countHerman}</span>
          </label>

          <label class="flex items-center gap-2 cursor-pointer select-none">
            <input type="checkbox" bind:checked={filterSyver} class="w-4 h-4 rounded accent-gray-500" />
            <span class="text-sm font-semibold text-gray-700">Syver</span>
            <span class="text-xs font-bold bg-gray-100 text-gray-600 rounded-full px-2 py-0.5">{countSyver}</span>
          </label>

        </div>
        {#if filterHerman || filterSyver}
          <p class="text-xs text-gray-400 mt-2">Viser kun økter med {[filterHerman && 'Herman', filterSyver && 'Syver'].filter(Boolean).join(' eller ')}</p>
        {/if}
      </div>
      {/if}

    </div>

    <!-- ── Calendar ─────────────────────────────────────────────────── -->

    <!-- DESKTOP: grid calendar (hidden on mobile) -->
    <div class="hidden sm:block bg-white rounded-xl border border-gray-200 overflow-hidden shadow-sm">

      <!-- Day headers -->
      <div class="grid grid-cols-7 border-b border-gray-200 bg-gray-50">
        {#each NO_DAYS_SHORT as day}
          <div class="py-3 text-center text-xs font-bold text-gray-500 uppercase tracking-wider">{day}</div>
        {/each}
      </div>

      <!-- Day cells -->
      <div class="grid grid-cols-7">
        {#each calDaysWithEvents as { date, cur, events }, i}
          {@const today = isToday(date)}
          <div
            class="relative min-h-36 border-b border-r border-gray-100 p-1.5
              {i % 7 === 6 ? 'border-r-0' : ''}"
          >
            <!-- Date number -->
            <div class="flex justify-end mb-1">
              <span class="text-sm leading-none w-6 h-6 flex items-center justify-center
                {today
                  ? 'bg-teal-600 text-white rounded-full font-bold'
                  : cur
                    ? 'text-gray-700 font-semibold'
                    : 'text-gray-400'}
              ">{date.getDate()}</span>
            </div>

            <!-- Event chips -->
            {#each events as evt}
              <button
                on:click={() => selectedEvent = evt}
                class="w-full text-left text-xs px-1.5 py-1 rounded mb-0.5
                  whitespace-normal break-words leading-snug min-h-9
                  font-medium transition hover:opacity-80 focus:outline-none
                  {GROUP[evt.group].chip}"
                title={evt.title}
              >{evt.title}</button>
            {/each}

            <!-- Off-month overlay -->
            {#if !cur}
              <div class="absolute inset-0 bg-gray-200/50 pointer-events-none"></div>
            {/if}

          </div>
        {/each}
      </div>

    </div><!-- /desktop calendar -->

    <!-- MOBILE: list view (hidden on sm+) -->
    <div class="sm:hidden bg-white rounded-xl border border-gray-200 overflow-hidden shadow-sm">
      {#each mobileDays as { date, events }, i}
        {@const today = isToday(date)}
        {@const dow = NO_DAYS_SHORT[(date.getDay() + 6) % 7]}
        {@const dateLabel = `${String(date.getDate()).padStart(2,'0')}.${String(date.getMonth()+1).padStart(2,'0')}`}
        <div class="{i > 0 ? 'border-t border-gray-100' : ''}">

          <!-- Day header row -->
          <div class="flex items-center gap-2 px-4 py-2
            {today ? 'bg-teal-50' : events.length > 0 ? 'bg-white' : 'bg-gray-50/60'}">
            <span class="text-sm font-black uppercase tracking-wider mr-2
              {today ? 'text-teal-600' : events.length > 0 ? 'text-gray-800' : 'text-gray-400'} w-7">{dow}</span>
            <span class="text-sm font-black
              {today ? 'text-teal-700' : events.length > 0 ? 'text-gray-800' : 'text-gray-400'}">{dateLabel}</span>
            {#if today}
              <span class="text-xs font-bold text-teal-600 bg-teal-100 rounded-full px-2 py-0.5 ml-1">I dag</span>
            {/if}
          </div>

          <!-- Events -->
          {#if events.length > 0}
            <div class="px-3 pb-3 space-y-1.5">
              {#each events as evt}
                <button
                  on:click={() => selectedEvent = evt}
                  class="w-full text-left text-sm px-3 py-2.5 rounded-lg
                    whitespace-normal break-words leading-snug
                    font-medium transition active:opacity-70 focus:outline-none
                    {GROUP[evt.group].chip}"
                  title={evt.title}
                >{evt.title}</button>
              {/each}
            </div>
          {/if}

        </div>
      {/each}
    </div><!-- /mobile list -->

  </main>
</div><!-- /page -->


<!-- ═══ Modal ══════════════════════════════════════════════════════════════ -->
{#if selectedEvent}
  <!-- svelte-ignore a11y-click-events-have-key-events -->
  <!-- svelte-ignore a11y-no-static-element-interactions -->
  <div
    class="fixed inset-0 bg-black/50 flex items-center justify-center z-50 p-4 backdrop-blur-sm"
    on:click|self={() => selectedEvent = null}
  >
    <div class="bg-white rounded-2xl overflow-hidden max-w-3xl w-full shadow-2xl flex max-h-[90vh]">

      <!-- ── Left panel ── -->
      <div class="relative overflow-hidden 5 flex-shrink-0 flex flex-col justify-between p-6 text-white {GROUP[selectedEvent.group].panel}">

        <!-- Bakgrunnstekst: gruppenavn -->
        <div class="absolute bottom-0 right-0 pointer-events-none select-none text-white/5 font-black uppercase overflow-hidden"
            style="font-size: 7rem; line-height: 0.80; margin-right: -1rem;">
          {#if selectedEvent.group === 'felles'}
            <div>HL</div>
            <div>JR</div>
          {:else}
            <div>{GROUP[selectedEvent.group].label}</div>
          {/if}
        </div>

        <!-- Top: label + big date -->
        <div>
          <p class="text-xs font-small uppercase tracking-widest opacity-60 mb-4">Treningsøkt</p>
          <p class="text-4xl font-black leading-none">{selectedEvent.date.getDate()}. {NO_MONTHS[selectedEvent.date.getMonth()]}</p>
        </div>

        <!-- Bottom: group + trainer -->
        <div class="space-y-3">
          <div class="flex items-center gap-2 text-sm">
            <strong> Gruppe: </strong> {GROUP[selectedEvent.group].label}
          </div>

          {#if selectedEvent.trainer}
            <div class="flex items-center gap-2 text-sm">
              <span> <strong> Trener: </strong> {selectedEvent.trainer}</span>
            </div>
          {/if}
        </div>
      </div>

      <!-- ── Right panel ── -->
      <div class="flex-1 flex flex-col overflow-hidden">

        <!-- Close button -->
        <div class="flex justify-end p-4 pb-0">
          <button
            on:click={() => selectedEvent = null}
            class="text-gray-400 hover:text-gray-700 transition"
            aria-label="Lukk"
          >
            <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/>
            </svg>
          </button>
        </div>

        <!-- Scrollable content -->
        <div class="flex-1 overflow-y-auto px-7 pb-7 pt-3 space-y-5">

          <!-- Title -->
          <h2 class="text-2xl font-black text-gray-900 leading-tight">
            {selectedEvent.title}
          </h2>

          <!-- Beskrivelse – full width -->
          {#if selectedEvent.description}
            <div class="pb-8">
              <p class="text-sm text-gray-700 leading-relaxed">{selectedEvent.description}</p>
            </div>
          {/if}

          <!-- Fokus + Oppmøtested side by side -->
          <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">

            {#if selectedEvent.focus}
              <div class="mb-4">
                <p class="text-xs font-small uppercase tracking-widest mb-2 {GROUP[selectedEvent.group].text}">Fokus</p>
                <div class="bg-gray-50 border border-gray-100 rounded-xl p-4 text-sm text-gray-700 leading-relaxed h-full">
                  {selectedEvent.focus}
                </div>
              </div>
            {/if}

            {#if selectedEvent.location}
              <div>
                <p class="text-xs font-small uppercase tracking-widest mb-2 {GROUP[selectedEvent.group].text}">Oppmøtested</p>
                <div class="flex items-start gap-2.5">
                  <div class="w-8 h-8 rounded-lg bg-gray-100 flex items-center justify-center flex-shrink-0 mt-0.5">
                    <svg class="w-4 h-4 text-gray-500" fill="currentColor" viewBox="0 0 20 20">
                      <path fill-rule="evenodd" d="M5.05 4.05a7 7 0 119.9 9.9L10 18.9l-4.95-4.95a7 7 0 010-9.9zM10 11a2 2 0 100-4 2 2 0 000 4z" clip-rule="evenodd"/>
                    </svg>
                  </div>
                  <div>
                    <p class="font-bold text-gray-900 text-sm">{selectedEvent.location}</p>
                  </div>
                </div>
              </div>
            {/if}

          </div>

          <!-- Kart – full bredde -->
          {#if mapUrl}
            <div class="rounded-xl overflow-hidden border border-gray-200 shadow-sm">
              <iframe
                src={mapUrl}
                width="100%"
                height="250"
                style="border:0; display:block;"
                allowfullscreen=""
                loading="lazy"
                referrerpolicy="no-referrer-when-downgrade"
                title="Kart over {selectedEvent.location}"
              ></iframe>
            </div>
          {/if}
        </div>
      </div>

    </div>
  </div>
{/if}

<!-- ═══ Admin Panel ═══════════════════════════════════════════════════════════ -->

{/if}
