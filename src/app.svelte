<script lang="ts">
  import { tick } from "svelte";
  import { fade, fly } from "svelte/transition";
  import { cubicOut, cubicInOut } from "svelte/easing";

  const DEFAULT_BANNER =
    "https://media.discordapp.net/attachments/1435378518056898642/1439403169493811221/b03d76a26e01b2a887f165bbfd9777f9.png?ex=691a642a&is=691912aa&hm=daa88888bd981706a34a4be2bea2aa23b78415446ef170ffcc611f604e632d98&=&format=webp&quality=lossless";
  const BANNERS: string[] = [
    DEFAULT_BANNER,
    "https://media.discordapp.net/attachments/1435378518056898642/1439401418258645144/download_2.gif?ex=691a6288&is=69191108&hm=a7a2ca7e6dcb3414e610b3eedd7ba8301131d6a05709ebb2dc61d44daed3f56a&=",
    "https://media.discordapp.net/attachments/1435378518056898642/1439399993650839714/download_1.gif?ex=691a6135&is=69190fb5&hm=a80e164f829aadc92e070f58ca365158d89887874f4823ba6c80e9d5cc574ae4&=&width=517&height=646",
    "https://media.discordapp.net/attachments/1435378518056898642/1439135774523002931/download.gif?ex=69196b22&is=691819a2&hm=3bf799535c9fc608d2b6ccbdf78c9333bb63935b3a6a54c991271b99feabefb5&=width=676&height=380",
  ];

  type ThemeVars = { [k: string]: string };
  type Theme = { name: string; vars: ThemeVars };
  const THEMES: Theme[] = [
    {
      name: "Black",
      vars: {
        "--bg-900": "rgba(10,10,12,.94)",
        "--bg-850": "rgba(8,8,10,.90)",
        "--bg-800": "rgba(16,16,20,.82)",
        "--bg-750": "rgba(22,22,26,.72)",
        "--text": "#f1f1f3",
        "--text-dim": "rgba(241,241,243,.80)",
        "--accent": "#7a4dff",
        "--accent-600": "#5e3de6",
        "--accent-soft": "rgba(122,77,255,.22)",
      },
    },
    {
      name: "Dark Red",
      vars: {
        "--bg-900": "rgba(18,6,8,.94)",
        "--bg-850": "rgba(14,6,8,.90)",
        "--bg-800": "rgba(26,10,12,.82)",
        "--bg-750": "rgba(34,12,14,.72)",
        "--text": "#f7eaea",
        "--text-dim": "rgba(247,234,234,.82)",
        "--accent": "#b02a2a",
        "--accent-600": "#8f1f1f",
        "--accent-soft": "rgba(176,42,42,.22)",
      },
    },
    {
      name: "Custom",
      vars: {
        "--bg-900": "rgba(10,12,14,.94)",
        "--bg-850": "rgba(8,10,12,.90)",
        "--bg-800": "rgba(14,18,22,.82)",
        "--bg-750": "rgba(20,26,30,.72)",
        "--text": "#eef7f9",
        "--text-dim": "rgba(238,247,249,.80)",
        "--accent": "#2bbbad",
        "--accent-600": "#249e93",
        "--accent-soft": "rgba(43,187,173,.22)",
      },
    },
  ];

  let current = 0;
  let activeMenu: any[] = [];
  let activeTabs = ["Main Menu"];
  let currentTab = 0;
  let showing = true;

  let itemRefs: HTMLElement[] = [];
  let textRefs: HTMLInputElement[] = [];
  let position: string = "left";
  let offsetX = 0;
  let offsetY = 0;

  let banner: string = DEFAULT_BANNER;
  let bannerOk = true;

  let themeIndex = 0;
  function applyThemeVars(vars: Record<string, string>) {
    const root = document.documentElement;
    for (const [k, v] of Object.entries(vars)) {
      root.style.setProperty(k, v);
    }
  }
  function applyThemeIndex(idx: number) {
    const clamped = Math.max(0, Math.min(idx | 0, THEMES.length - 1));
    themeIndex = clamped;
    applyThemeVars(THEMES[themeIndex]?.vars || {});
  }

  let wrapperEl: HTMLElement;
  let listEl: HTMLElement;
  let trackTop = 0,
    trackHeight = 0,
    thumbTop = 0,
    thumbHeight = 24;
  let hasOverflow = false;
  const MIN_THUMB = 18,
    NO_OVERFLOW_THUMB_MAX = 28;
  let capturingIndex: number | null = null;

  // Hover pop-out (Lua->NUI)
  let popVisible = false;
  let popName = "";
  let popSid: number | null = null;
  let popDist = 0;
  let popHealth = 0;
  let popArmour = 0;
  let popVehicle = "";
  let popInVeh = false;
  let popAlive = true;
  let popSpeedKmh = 0;
  let popIsVisible = true;
  let popWeaponName = "";

  // Spectator overlay (World -> Spectator Watch)
  let spectatorBox: { show: boolean; count: number } = {
    show: false,
    count: 0,
  };

  // Open-key prompt (host-controlled, hidden by default)
  let keyPrompt = { show: false, vk: null as number | null, label: "None" };

  // Derived selectable list
  let selectableMenu: any[] = [];
  $: selectableMenu = activeMenu.filter((o) => o?.type !== "divider");
  let selectableIndices: number[] = [];
  $: selectableIndices = activeMenu
    .map((o, i) => (o?.type !== "divider" ? i : -1))
    .filter((i) => i !== -1);

  function counterCurrent(): number {
    const total = selectableIndices.length;
    if (total === 0) return 0;
    const pos = selectableIndices.indexOf(current);
    if (pos !== -1) return pos + 1;
    let nearest = -1,
      best = 1e9;
    for (let i = 0; i < total; i++) {
      const d = Math.abs(selectableIndices[i] - current);
      if (d < best) {
        best = d;
        nearest = i;
      }
    }
    return nearest !== -1 ? nearest + 1 : 0;
  }

  function syncThumb() {
    if (!listEl) return;
    const trackScrollable = Math.max(0, trackHeight - thumbHeight);
    if (hasOverflow) {
      const maxScroll = Math.max(0, listEl.scrollHeight - listEl.clientHeight);
      const ratio =
        maxScroll > 0
          ? Math.min(1, Math.max(0, listEl.scrollTop / maxScroll))
          : 0;
      thumbTop = Math.round(trackScrollable * ratio);
    } else {
      const count = Math.max(1, activeMenu.length);
      const idx = Math.min(Math.max(current, 0), count - 1);
      const last = Math.max(1, count - 1);
      const ratio = count > 1 ? idx / last : 0;
      thumbTop = Math.round(trackScrollable * ratio);
    }
  }

  function measureScrollbar() {
    if (!wrapperEl || !listEl) return;
    const wrapperRect = wrapperEl.getBoundingClientRect();
    const listRect = listEl.getBoundingClientRect();
    const clientH = listEl.clientHeight;
    const scrollH = listEl.scrollHeight;
    hasOverflow = scrollH > clientH + 1;
    trackTop = Math.round(listRect.top - wrapperRect.top);
    const contentH = Math.min(clientH, scrollH || clientH);
    trackHeight = Math.round(hasOverflow ? clientH : contentH);
    thumbHeight = hasOverflow
      ? Math.max(MIN_THUMB, Math.round((clientH / scrollH) * trackHeight))
      : Math.max(MIN_THUMB, Math.min(NO_OVERFLOW_THUMB_MAX, trackHeight));
    syncThumb();
  }

  let measureReq = 0;
  function scheduleMeasure() {
    if (measureReq) return;
    measureReq = requestAnimationFrame(() => {
      measureReq = 0 as any;
      measureScrollbar();
    });
  }

  function animateScrollTo(el: HTMLElement, target: number, duration = 240) {
    const start = el.scrollTop;
    const distance = target - start;
    if (Math.abs(distance) < 1) {
      el.scrollTop = target;
      syncThumb();
      return;
    }
    const startTime = performance.now();
    function step(now: number) {
      const t = Math.min(1, (now - startTime) / duration);
      const eased = cubicInOut(t);
      el.scrollTop = start + distance * eased;
      syncThumb();
      if (t < 1) requestAnimationFrame(step);
    }
    requestAnimationFrame(step);
  }

  function centerOnSelected(index: number) {
    if (!listEl) return;
    const maxScroll = Math.max(0, listEl.scrollHeight - listEl.clientHeight);
    if (!hasOverflow || maxScroll === 0) {
      animateScrollTo(listEl, 0);
      return;
    }
    const lastIdx = Math.max(0, activeMenu.length - 1);
    if (index <= 0) {
      animateScrollTo(listEl, 0);
      return;
    }
    if (index >= lastIdx) {
      animateScrollTo(listEl, maxScroll);
      return;
    }
    const item = itemRefs[index];
    if (!item) return;
    const itemTop = item.offsetTop;
    const itemCenter = itemTop + item.offsetHeight / 2;
    let target = itemCenter - listEl.clientHeight / 2;
    target = Math.max(0, Math.min(maxScroll, target));
    animateScrollTo(listEl, target);
  }

  let scrollRAF = 0 as any;
  function onListScroll() {
    if (scrollRAF) return;
    scrollRAF = requestAnimationFrame(() => {
      scrollRAF = 0;
      syncThumb();
    });
  }
  $: if (showing) {
    centerOnSelected(current);
  }

  async function focusTextAt(idx: number) {
    await tick();
    textRefs[idx]?.focus();
  }

  function post(action: string, payload: any = {}) {
    const msg = JSON.stringify({ action, ...payload });
    try {
      window.postMessage(msg, "*");
    } catch {}
    try {
      (window.parent ?? window).postMessage(msg, "*");
    } catch {}
  }
  function commitText(index: number, value: string) {
    post("textCommit", { index, value });
  }

  function handleTextKeydown(index: number, option: any, e: KeyboardEvent) {
    e.stopPropagation();
    if (e.key === "Enter") {
      e.preventDefault();
      commitText(index, option.value ?? "");
      (e.currentTarget as HTMLInputElement)?.blur();
    } else if (e.key === "Escape") {
      e.preventDefault();
      (e.currentTarget as HTMLInputElement)?.blur();
    }
  }

  function vkFromEvent(e: KeyboardEvent): number | null {
    const key = e.key,
      code = e.code;
    const map: Record<string, number> = {
      Escape: 0x1b,
      Esc: 0x1b,
      Tab: 0x09,
      CapsLock: 0x14,
      Shift: 0x10,
      ShiftLeft: 0x10,
      ShiftRight: 0x10,
      Control: 0x11,
      ControlLeft: 0x11,
      ControlRight: 0x11,
      Alt: 0x12,
      AltLeft: 0x12,
      AltRight: 0x12,
      Enter: 0x0d,
      NumpadEnter: 0x0d,
      Backspace: 0x08,
      Space: 0x20,
      " ": 0x20,
      Insert: 0x2d,
      Delete: 0x2e,
      Home: 0x24,
      End: 0x23,
      PageUp: 0x21,
      PageDown: 0x22,
      ArrowLeft: 0x25,
      ArrowUp: 0x26,
      ArrowRight: 0x27,
      ArrowDown: 0x28,
      Meta: 0x5b,
      OS: 0x5b,
      MetaLeft: 0x5b,
      MetaRight: 0x5c,
      NumpadMultiply: 0x6a,
      NumpadAdd: 0x6b,
      NumpadSubtract: 0x6d,
      NumpadDecimal: 0x6e,
      NumpadDivide: 0x6f,
      Semicolon: 0xba,
      ";": 0xba,
      Equal: 0xbb,
      "=": 0xbb,
      Comma: 0xbc,
      ",": 0xbc,
      Minus: 0xbd,
      "-": 0xbd,
      Period: 0xbe,
      ".": 0xbe,
      Slash: 0xbf,
      "/": 0xbf,
      Backquote: 0xc0,
      "`": 0xc0,
      BracketLeft: 0xdb,
      "[": 0xdb,
      Backslash: 0xdc,
      "\\": 0xdc,
      BracketRight: 0xdd,
      "]": 0xdd,
      Quote: 0xde,
      "'": 0xde,
    };
    if (/^F\d{1,2}$/.test(key)) {
      const n = parseInt(key.slice(1), 10);
      if (n >= 1 && n <= 24) return 0x70 + (n - 1);
    }
    if (key && key.length === 1 && /^[A-Za-z]$/.test(key))
      return key.toUpperCase().charCodeAt(0);
    if (key && key.length === 1 && /^[0-9]$/.test(key))
      return 0x30 + parseInt(key, 10);
    if (/^Numpad[0-9]$/.test(code))
      return 0x60 + parseInt(code.replace("Numpad", ""), 10);
    return map[key] ?? map[code] ?? null;
  }
  function vkLabel(vk: number | null): string {
    if (vk == null) return "None";
    return "0x" + vk.toString(16).toUpperCase().padStart(2, "0");
  }

  function onWindowKeydown(e: KeyboardEvent) {
    if (capturingIndex !== null) {
      if (e.key === "Escape") {
        e.preventDefault();
        post("keyCaptured", { index: capturingIndex, vk: null });
        capturingIndex = null;
        return;
      }
      const vk = vkFromEvent(e);
      if (vk != null) {
        e.preventDefault();
        post("keyCaptured", { index: capturingIndex, vk });
        capturingIndex = null;
      }
      return;
    }
    if (keyPrompt.show) {
      if (e.key === "Enter" && keyPrompt.vk != null) {
        e.preventDefault();
        const vk = keyPrompt.vk;
        post("setOpenKey", { vk });
        keyPrompt.show = false;
        return;
      }
      const vk = vkFromEvent(e);
      if (vk != null) {
        e.preventDefault();
        keyPrompt.vk = vk;
        keyPrompt.label = vkLabel(vk);
      }
      return;
    }
  }
  window.addEventListener("keydown", onWindowKeydown);
  window.addEventListener("resize", scheduleMeasure);

  let lastHoverSid: number | null = null;
  $: if (showing) {
    const it: any = activeMenu?.[current];
    const sid = it && it.sid != null ? Number(it.sid) : null;
    if (sid !== lastHoverSid) {
      if (sid != null && !Number.isNaN(sid)) post("playerHoverStart", { sid });
      else post("playerHoverEnd");
      lastHoverSid = sid;
    }
  }

  type Toast = {
    id: number;
    title: string;
    message: string;
    ntype: "info" | "success" | "warning" | "error";
    icon?: string;
    timeout: number;
    progress?: number;
    paused?: boolean;
    timeoutId?: any;
    startTime?: number;
    remainingTime?: number;
  };
  let toasts: Toast[] = [];
  let nextToastId = 1;
  const MAX_TOASTS = 5;
  const defaultsByType = {
    info: {
      color: "var(--accent)",
      icon: "ph-info",
      gradient:
        "linear-gradient(135deg, var(--accent) 0%, var(--accent-600) 100%)",
    },
    success: {
      color: "var(--accent)",
      icon: "ph-check-circle",
      gradient:
        "linear-gradient(135deg, var(--accent) 0%, var(--accent-600) 100%)",
    },
    warning: {
      color: "var(--accent)",
      icon: "ph-warning",
      gradient:
        "linear-gradient(135deg, var(--accent) 0%, var(--accent-600) 100%)",
    },
    error: {
      color: "var(--accent)",
      icon: "ph-x-circle",
      gradient:
        "linear-gradient(135deg, var(--accent) 0%, var(--accent-600) 100%)",
    },
  } as const;

  function pushToast(t: Partial<Toast>) {
    const ntype = (t.ntype ?? "info") as Toast["ntype"];
    const def = defaultsByType[ntype];
    const timeout = Number(t.timeout ?? 4000);
    const toast: Toast = {
      id: nextToastId++,
      title: String(t.title ?? ""),
      message: String(t.message ?? ""),
      ntype,
      icon: String(t.icon ?? def.icon),
      timeout,
      progress: 100,
      paused: false,
      startTime: Date.now(),
      remainingTime: timeout,
    };
    toasts = [...toasts, toast];
    if (toasts.length > MAX_TOASTS)
      toasts = toasts.slice(toasts.length - MAX_TOASTS);

    startToastTimer(toast);
  }

  function startToastTimer(toast: Toast) {
    const startTime = Date.now();
    const duration = toast.remainingTime!;

    function updateProgress() {
      const t = toasts.find((x) => x.id === toast.id);
      if (!t) return;

      if (t.paused) {
        requestAnimationFrame(updateProgress);
        return;
      }

      const elapsed = Date.now() - startTime;
      const remaining = Math.max(0, duration - elapsed);
      const progress = (remaining / duration) * 100;

      t.progress = progress;
      t.remainingTime = remaining;
      toasts = [...toasts];

      if (remaining > 0) {
        requestAnimationFrame(updateProgress);
      } else {
        closeToast(toast.id);
      }
    }

    requestAnimationFrame(updateProgress);
  }

  function pauseToast(id: number) {
    const toast = toasts.find((t) => t.id === id);
    if (toast) {
      toast.paused = true;
      toasts = [...toasts];
    }
  }

  function resumeToast(id: number) {
    const toast = toasts.find((t) => t.id === id);
    if (toast && toast.paused) {
      toast.paused = false;
      toasts = [...toasts];
    }
  }

  function closeToast(id: number) {
    toasts = toasts.filter((t) => t.id !== id);
  }

  window.addEventListener("message", (event) => {
    let payload: any = event.data ?? {};
    if (typeof payload === "string") {
      try {
        payload = JSON.parse(payload);
      } catch {
        return;
      }
    }
    const { action, ...data } = payload;
    if (action === "setCurrent") {
      current = (data.current || 1) - 1;
      activeMenu = data.menu || [];
      showing = true;
      scheduleMeasure();
      tick().then(() => centerOnSelected(current));
    } else if (action === "setTabs") {
      activeTabs =
        Array.isArray(data.tabs) && data.tabs.length
          ? data.tabs
          : ["Main Menu"];
      currentTab = 0;
    } else if (action === "setTabIndex") {
      currentTab = data.index || 0;
    } else if (action === "setVisible") {
      showing = !!data.visible;
      if (showing) {
        scheduleMeasure();
        tick().then(() => centerOnSelected(current));
      } else {
        popVisible = false;
      }
    } else if (action === "position") {
      position = data.position || "left";
      scheduleMeasure();
    } else if (action === "menuOffsets") {
      offsetX = Number(data.x) || 0;
      offsetY = Number(data.y) || 0;
      scheduleMeasure();
    } else if (action === "banner") {
      bannerOk = true;
      banner = data.banner || DEFAULT_BANNER;
    } else if (action === "setBanner") {
      bannerOk = true;
      banner = data.url || data.banner || DEFAULT_BANNER;
    } else if (action === "bannerIndex" || action === "setBannerIndex") {
      const idx = Math.max(
        0,
        Math.min(Number(data.index) || 0, BANNERS.length - 1),
      );
      bannerOk = true;
      banner = BANNERS[idx] || DEFAULT_BANNER;
    } else if (action === "themeIndex" || action === "setThemeIndex") {
      applyThemeIndex(Number(data.index) || 0);
    } else if (action === "beginTextEdit") {
      focusTextAt(Number(data.index) || 0);
    } else if (action === "beginKeyCapture") {
      capturingIndex = Number(data.index) || 0;
    } else if (action === "RebuildPlayerPopOutFinderUI") {
      popVisible = !!data.visible;
      if (popVisible) {
        if (data.sid !== undefined) popSid = Number(data.sid);
        if (data.name !== undefined) popName = String(data.name ?? "");
        if (data.dist !== undefined) popDist = Number(data.dist ?? 0);
        if (data.health !== undefined) popHealth = Number(data.health ?? 0);
        if (data.armour !== undefined) popArmour = Number(data.armour ?? 0);
        if (data.vehicle !== undefined) popVehicle = String(data.vehicle ?? "");
        if (data.inVeh !== undefined) popInVeh = !!data.inVeh;
        if (data.alive !== undefined) popAlive = !!data.alive;
        if (data.speedKmh !== undefined)
          popSpeedKmh = Number(data.speedKmh ?? 0);
        if (data.isVisible !== undefined) popIsVisible = !!data.isVisible;
        if (data.weaponName !== undefined)
          popWeaponName = String(data.weaponName ?? "");
      }
    } else if (action === "showKeyPrompt") {
      keyPrompt.show = !!data.show;
      if (keyPrompt.show) {
        keyPrompt.vk = null;
        keyPrompt.label = "None";
      }
    } else if (action === "keyPromptDone") {
      keyPrompt.show = false;
    } else if (action === "setSpectatorOverlay") {
      spectatorBox.show = !!data.enabled;
      spectatorBox.count = Number(data.count ?? 0);
    } else if (action === "notify") {
      pushToast(data);
    }
  });

  function onRowEnter(opt: any) {
    if (opt && opt.sid != null) post("playerHoverStart", { sid: opt.sid });
  }
  function onRowLeave(opt: any) {
    if (opt && opt.sid != null) post("playerHoverEnd");
  }

  applyThemeIndex(themeIndex);
</script>

<svelte:head>
  <script src="https://unpkg.com/@phosphor-icons/web"></script>
</svelte:head>

{#if showing}
  <div
    bind:this={wrapperEl}
    class="menu-root"
    style="position:absolute; top:calc(50% + {offsetY}vh); transform:translate({offsetX}vh, -50%); left:{position ==
    'left'
      ? '6.5vh'
      : 'calc(100% - 41vh)'};"
  >
    <div
      class="scrollbar-container"
      style="top:{trackTop}px; height:{trackHeight}px;"
    >
      <div
        class="scrollbar-thumb"
        style="top:{thumbTop}px; height:{thumbHeight}px;"
      />
    </div>
    <div
      style="transform: translateY(-0.6vh); margin-left:1.75vh; width:95%; border-radius:.4vh; overflow:hidden; line-height:0; box-shadow:0 .4vh 1.6vh rgba(0,0,0,.4);"
    >
      {#if banner && bannerOk}
        <img
          src={banner}
          crossorigin="anonymous"
          on:error={() => (bannerOk = false)}
          alt="Banner"
          style="display:block; width:100%; height:9.5vh; object-fit:cover; margin:0; padding:0; border:0; will-change: opacity;"
          in:fade={{ duration: 240 }}
        />
      {:else}
        <div
          style="width:100%; height:9.5vh; background:linear-gradient(90deg, rgba(122,77,255,.35), rgba(122,77,255,.05));"
        ></div>
      {/if}
      <div
        style="display:flex; align-items:center; justify-content:center; background:var(--bg-850); border-radius:0 0 .4vh .4vh;"
        in:fade={{ duration: 160 }}
      >
        {#if activeTabs.length === 1}
          <div
            class="banner-title"
            style="flex:1; text-align:center; padding:1.5vh 1.5vh; font-size:1.15vh; font-weight:600; letter-spacing:.03em; background:linear-gradient(0deg, var(--accent-600) 0%, transparent 100%); color:var(--text); white-space:nowrap;"
          >
            {activeTabs[0]}
          </div>
        {:else}
          {#each activeTabs as tab, i (tab)}
            <div
              class="banner-title"
              style="flex:1; text-align:center; padding:1.5vh 1.5vh; font-size:1.15vh; font-weight:500; cursor:pointer; transition:all .15s ease; background:{i ===
              currentTab
                ? 'linear-gradient(0deg, var(--accent-600) 0%, transparent 100%)'
                : 'transparent'}; color:var(--text);"
              in:fade={{ duration: 140 }}
            >
              {tab}
            </div>
          {/each}
        {/if}
      </div>
    </div>
    <div
      class="list-shell"
      style="position:absolute; margin-left:1.75vh; width:33.5vh; background:var(--bg-900); border-radius:.4vh; display:flex; flex-direction:column; overflow:hidden; position:relative; box-shadow:0 .5vh 1.8vh rgba(0,0,0,.4);"
    >
      <div
        class="hide-scrollbar"
        bind:this={listEl}
        on:scroll={onListScroll}
        style="max-height:33.6vh; content-visibility:auto; contain-intrinsic-size: 300px 420px;"
      >
        <div
          style="display:flex; flex-direction:column; gap:0; position:relative;"
        >
          {#each activeMenu as option, index (option.label)}
            {#if option.type == "divider"}
              <div class="menu-divider" in:fade={{ duration: 120 }}>
                <span>{option.label}</span>
              </div>
            {:else}
              <div
                bind:this={itemRefs[index]}
                class="menu-item {index == current ? 'active' : ''}"
                style="padding:.8vh 1.0vh; color:var(--text); border-radius:.3vh; justify-content:space-between; display:flex; align-items:center; font-size:1.4vh; gap:1vh; font-weight:400; cursor:pointer; z-index:1;"
                on:mouseenter={() => onRowEnter(option)}
                on:mouseleave={() => onRowLeave(option)}
              >
                <span
                  style="flex-grow:1; text-align:left; font-size:1.2vh; height:100%;"
                  >{option.label}</span
                >
                {#if option.type == "text"}
                  <input
                    bind:this={textRefs[index]}
                    class="text-input"
                    type="text"
                    bind:value={option.value}
                    placeholder={option.placeholder || "Type..."}
                    on:keydown={(e) => handleTextKeydown(index, option, e)}
                    on:blur={() => commitText(index, option.value ?? "")}
                  />
                {:else if option.type == "keybind"}
                  <span
                    class="key-pill {capturingIndex === index
                      ? 'capture-pill'
                      : ''}"
                    >{capturingIndex === index
                      ? "Press any key..."
                      : option.vk != null
                        ? `VK ${option.vk}`
                        : "None"}</span
                  >
                {:else if option.type == "submenu"}
                  <i
                    class="ph ph-caret-right"
                    style="color:var(--text); font-size:1.2vh;"
                  ></i>
                {:else if option.type == "checkbox"}
                  <div class="toggle {option.checked ? 'on' : ''}"></div>
                {:else if option.type == "slider"}
                  <input
                    type="range"
                    class="range"
                    bind:value={option.value}
                    min={option.min}
                    max={option.max}
                    step={option.step}
                    style="--pct:{(Number(option.value || 0) /
                      Number(option.max || 1)) *
                      100}%;"
                  />
                {:else if option.type == "scroll"}
                  <div
                    style="position:relative; display:flex; align-items:center; justify-content:center; width:auto; height:100%; gap:.4vh; color:var(--text);"
                  >
                    <span style="font-size:1.2vh; line-height:1;">-</span>
                    <span style="font-size:1.2vh; line-height:1;"
                      >{option.options?.[option.selected]?.label}</span
                    >
                    <span style="font-size:1.2vh; line-height:1;">-</span>
                  </div>
                {/if}
              </div>
            {/if}
          {/each}
        </div>
      </div>
    </div>
    {#if popVisible}
      <div
        class="player-popout"
        in:fly={{ x: 16, duration: 160, easing: cubicOut }}
        out:fade={{ duration: 120 }}
      >
        <div class="pop-title">{popName} [{popSid}]</div>
        <div class="kv">
          <span class="label">Distance</span><strong class="value"
            >{(Number(popDist) || 0).toFixed(1)} m</strong
          >
        </div>
        <div class="kv">
          <span class="label">Server ID</span><strong class="value"
            >{popSid}</strong
          >
        </div>
        <div class="kv">
          <span class="label">Health</span><strong class="health value"
            >{Math.round(Number(popHealth) || 0)}</strong
          >
        </div>
        <div class="bar">
          <div
            class="fill"
            style={`width:${Math.max(0, Math.min(100, Number(popHealth) || 0))}%;`}
          />
        </div>
        <div class="kv">
          <span class="label">Armour</span><strong class="value"
            >{Math.round(Number(popArmour) || 0)}</strong
          >
        </div>
        <div class="kv">
          <span class="label">Weapon</span><strong class="value"
            >{popWeaponName || "Unknown"}</strong
          >
        </div>
        <div class="kv">
          <span class="label">Vehicle</span><strong class="value"
            >{popInVeh ? popVehicle || "In Vehicle" : "On Foot"}</strong
          >
        </div>
        <div class="kv">
          <span class="label">Alive</span><strong class="value"
            >{popAlive ? "Alive" : "Dead"}</strong
          >
        </div>
        <div class="kv">
          <span class="label">Speed</span><strong class="value"
            >{Math.round(Number(popSpeedKmh) || 0)} km/h</strong
          >
        </div>
        <div class="kv">
          <span class="label">Visible</span><strong class="value"
            >{popIsVisible ? "Yes" : "No"}</strong
          >
        </div>
        <div class="hint">
          Open a player entry to Spectate, Inventory, or Go To.
        </div>
      </div>
    {/if}
    <div
      class="footer-bar"
      style="margin-top:.5vh; margin-left:1.75vh; width:90%; background:var(--bg-800); border-radius:.3vh; padding:.8vh .9vh; font-size:1.1vh; color:var(--text); display:flex; justify-content:space-between; align-items:center; box-shadow:0 .3vh 1.2vh rgba(0,0,0,.35);"
      in:fly={{ y: 8, duration: 200, easing: cubicOut }}
    >
      <span style="color:var(--text); font-weight:500;"
        >Beta | CapTcha @2025</span
      >
      <span style="color:var(--text); font-weight:500;"
        >({selectableMenu.length
          ? counterCurrent()
          : 0}/{selectableMenu.length})</span
      >
    </div>
  </div>
{/if}

{#if keyPrompt.show}
  <div
    class="openkey-pop"
    in:fly={{ x: 12, duration: 140 }}
    out:fade={{ duration: 120 }}
  >
    <div class="t">Choose open key</div>
    <div class="v">Current: <strong>{keyPrompt.label}</strong></div>
    <div class="h">Press a key, then Enter to confirm</div>
  </div>
{/if}

{#if spectatorBox.show}
  <div class="spectator-box">
    <div class="spec-title">SPECTATOR</div>
    <div class="spec-count">
      {Math.max(0, Math.floor(spectatorBox.count || 0))}
    </div>
  </div>
{/if}

<div class="noti-container">
  {#each toasts as t, index (t.id)}
    {@const def = defaultsByType[t.ntype]}
    <div
      class="noti-card"
      role="alert"
      aria-live="polite"
      style="--noti-color: {def.color}; --noti-gradient: {def.gradient}; --index: {index};"
      in:fly={{ x: 40, duration: 300, easing: cubicOut }}
      out:fly={{ x: 40, opacity: 0, duration: 250, easing: cubicOut }}
      on:mouseenter={() => pauseToast(t.id)}
      on:mouseleave={() => resumeToast(t.id)}
    >
      <div class="noti-progress" style="width: {t.progress || 0}%;" />
      <div class="noti-glow" />
      <div class="noti-content">
        <div class="noti-body">
          <div class="noti-title">{t.title}</div>
          {#if t.message}
            <div class="noti-message">{t.message}</div>
          {/if}
        </div>
      </div>
    </div>
  {/each}
</div>

<style>
  :root {
    --accent: #6b4eff;
    --accent-600: #6b4eff;
    --accent-soft: rgba(122, 77, 255, 0.16);
    --bg-900: rgba(26, 19, 38, 0.92);
    --bg-850: rgba(8, 6, 12, 0.88);
    --bg-800: rgba(0, 0, 0, 0.8);
    --bg-750: rgba(0, 0, 0, 0.7);
    --text: #ffffff;
    --text-dim: rgba(255, 255, 255, 0.8);
  }
  * {
    background: none;
  }
  body {
    margin: 0;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
      system-ui, sans-serif;
  }

  /* Advanced Notification System */
  .noti-container {
    position: fixed;
    top: 2.5vh;
    right: 2.5vh;
    display: flex;
    flex-direction: column;
    gap: 1.2vh;
    z-index: 99999;
    pointer-events: none;
  }

  .noti-card {
    pointer-events: auto;
    position: relative;
    width: 32vh;
    background: transparent;
    border-radius: 1.2vh;
    overflow: visible;
    transform-origin: right center;
    animation: notiSlideIn 0.4s cubic-bezier(0.16, 1, 0.3, 1);
    transition: all 0.3s cubic-bezier(0.16, 1, 0.3, 1);
  }

  .noti-card::before {
    content: "";
    position: absolute;
    inset: 0;
    background: linear-gradient(
      135deg,
      rgba(18, 18, 26, 0.96) 0%,
      rgba(12, 12, 18, 0.98) 100%
    );
    border-radius: 1.2vh;
    border: 0.15vh solid rgba(255, 255, 255, 0.15);
    box-shadow:
      0 0.8vh 2.5vh rgba(0, 0, 0, 0.5),
      0 0.3vh 1vh rgba(0, 0, 0, 0.3),
      inset 0 1px 0 rgba(255, 255, 255, 0.08);
    z-index: -1;
  }

  .noti-card:hover {
    transform: translateX(-0.5vh) scale(1.02);
  }

  .noti-card:hover::before {
    background: linear-gradient(
      135deg,
      rgba(22, 22, 32, 0.98) 0%,
      rgba(16, 16, 24, 1) 100%
    );
    border-color: rgba(255, 255, 255, 0.25);
    box-shadow:
      0 1vh 3vh rgba(0, 0, 0, 0.6),
      0 0.5vh 1.5vh rgba(0, 0, 0, 0.4),
      0 0 3vh var(--noti-color),
      inset 0 1px 0 rgba(255, 255, 255, 0.12);
  }

  .noti-progress {
    position: absolute;
    bottom: 0;
    left: 0;
    height: 0.4vh;
    background: var(--noti-gradient);
    transition: width 0.1s linear;
    box-shadow: 0 0 1vh var(--noti-color);
    z-index: 2;
  }

  .noti-glow {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    height: 100%;
    background: radial-gradient(
      circle at 20% 20%,
      var(--noti-color),
      transparent 70%
    );
    opacity: 0.04;
    pointer-events: none;
    z-index: 0;
  }

  .noti-content {
    position: relative;
    display: flex;
    align-items: flex-start;
    gap: 1vh;
    padding: 1.2vh 1.4vh;
    z-index: 1;
  }

  .noti-icon-wrapper {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 3.5vh;
    height: 3.5vh;
    min-width: 3.5vh;
    background: var(--noti-gradient);
    border-radius: 1vh;
    box-shadow:
      0 0.5vh 1.5vh rgba(0, 0, 0, 0.3),
      inset 0 -0.2vh 0.5vh rgba(0, 0, 0, 0.2),
      0 0 1.5vh var(--noti-color);
    animation: notiIconPop 0.5s cubic-bezier(0.16, 1, 0.3, 1) 0.1s backwards;
  }

  .noti-icon-wrapper i {
    font-size: 1.8vh;
    color: #ffffff;
    filter: drop-shadow(0 0.2vh 0.4vh rgba(0, 0, 0, 0.3));
  }

  .noti-body {
    flex: 1;
    display: flex;
    flex-direction: column;
    gap: 0.5vh;
    min-width: 0;
  }

  .noti-title {
    font-size: 1.5vh;
    font-weight: 700;
    color: #ffffff;
    letter-spacing: 0.02em;
    line-height: 1.3;
    text-shadow: 0 0.1vh 0.3vh rgba(0, 0, 0, 0.5);
    animation: notiTextSlide 0.4s cubic-bezier(0.16, 1, 0.3, 1) 0.15s backwards;
  }

  .noti-message {
    font-size: 1.25vh;
    font-weight: 400;
    color: rgba(255, 255, 255, 0.85);
    line-height: 1.4;
    letter-spacing: 0.01em;
    animation: notiTextSlide 0.4s cubic-bezier(0.16, 1, 0.3, 1) 0.2s backwards;
  }

  .noti-close {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 2.8vh;
    height: 2.8vh;
    min-width: 2.8vh;
    background: rgba(255, 255, 255, 0.06);
    border: none;
    border-radius: 0.6vh;
    cursor: pointer;
    transition: all 0.2s cubic-bezier(0.16, 1, 0.3, 1);
    padding: 0;
    margin-top: 0.2vh;
  }

  .noti-close i {
    font-size: 1.6vh;
    color: rgba(255, 255, 255, 0.6);
    transition: color 0.2s ease;
  }

  .noti-close:hover {
    background: rgba(255, 255, 255, 0.12);
    transform: scale(1.1);
  }

  .noti-close:hover i {
    color: rgba(255, 255, 255, 0.95);
  }

  .noti-close:active {
    transform: scale(0.95);
  }

  @keyframes notiSlideIn {
    from {
      opacity: 0;
      transform: translateX(10vh) scale(0.9);
    }
    to {
      opacity: 1;
      transform: translateX(0) scale(1);
    }
  }

  @keyframes notiIconPop {
    0% {
      opacity: 0;
      transform: scale(0) rotate(-180deg);
    }
    50% {
      transform: scale(1.15) rotate(10deg);
    }
    100% {
      opacity: 1;
      transform: scale(1) rotate(0deg);
    }
  }

  @keyframes notiTextSlide {
    from {
      opacity: 0;
      transform: translateY(0.5vh);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }

  .toggle {
    appearance: none;
    -webkit-appearance: none;
    width: 3.6vh;
    height: 1.8vh;
    background-color: var(--bg-750);
    border-radius: 1.2vh;
    position: relative;
    cursor: pointer;
    transition: background-color 0.25s cubic-bezier(0.2, 0.8, 0.2, 1);
    box-shadow: inset 0 0 0 0.12vh rgba(0, 0, 0, 0.35);
  }
  .toggle::before {
    content: "";
    position: absolute;
    width: 1.3vh;
    height: 1.3vh;
    border-radius: 50%;
    background: #fff;
    top: 0.25vh;
    left: 0.25vh;
    transition: transform 0.22s cubic-bezier(0.2, 0.8, 0.2, 1);
    box-shadow: 0 0.1vh 0.25vh rgba(0, 0, 0, 0.35);
  }
  .toggle.on::before {
    background: var(--accent);
    transform: translateX(1.8vh);
  }

  .range {
    width: 9vh;
    height: 1.2vh;
    background: transparent;
    margin: 0;
    outline: none;
    border: 0;
  }
  .range,
  .range::-webkit-slider-thumb {
    -webkit-appearance: none;
  }
  .range::-webkit-slider-runnable-track {
    height: 0.6vh;
    border-radius: 1.2vh;
    background: linear-gradient(
      to right,
      rgba(255, 255, 255, 0.95) 0 var(--pct),
      rgb(60, 45, 86) var(--pct) 100%
    );
    transition: background 0.12s linear;
  }
  .range::-webkit-slider-thumb {
    width: 1.2vh;
    height: 1.2vh;
    margin-top: -0.32vh;
    border-radius: 50%;
    background: #fff;
    cursor: pointer;
    box-shadow: 0 0.1vh 0.25vh rgba(0, 0, 0, 0.35);
    transition: transform 0.15s cubic-bezier(0.2, 0.8, 0.2, 1);
    will-change: transform;
  }
  .range:active::-webkit-slider-thumb {
    transform: scale(1.06);
  }

  .hide-scrollbar {
    overflow-y: auto;
    scrollbar-width: none;
    -ms-overflow-style: none;
  }
  .hide-scrollbar::-webkit-scrollbar {
    display: none;
    width: 0;
    height: 0;
  }

  .scrollbar-container {
    position: absolute;
    left: 0.5vh;
    width: 0.7vh;
    border-radius: 0.2vh;
    background: var(--bg-900);
    pointer-events: none;
    z-index: 100;
    transition:
      height 0.2s ease,
      opacity 0.2s ease;
    overflow: hidden;
  }
  .scrollbar-thumb {
    position: absolute;
    left: 0;
    right: 0;
    top: 0;
    height: 1.2vh;
    border-radius: 0.2vh;
    background: var(--accent);
    transition:
      top 180ms cubic-bezier(0.2, 0.8, 0.2, 1),
      height 200ms ease;
    will-change: top, height;
    transform: translateZ(0);
  }

  .menu-divider {
    position: relative;
    display: flex;
    align-items: center;
    justify-content: center;
    height: 2.2vh;
    margin: 0.6vh 0;
    pointer-events: none;
  }
  .menu-divider span {
    color: var(--text-dim);
    font-size: 1vh;
    font-weight: 600;
    letter-spacing: 0.02em;
  }
  .menu-divider::before,
  .menu-divider::after {
    content: "";
    position: absolute;
    top: 50%;
    transform: translateY(-50%);
    width: 7vh;
    height: 0.4vh;
    background: var(--accent);
    border-radius: 0.3vh;
    opacity: 0.95;
  }
  .menu-divider::before {
    left: 1.2vh;
  }
  .menu-divider::after {
    right: 1.2vh;
  }

  .menu-item {
    transition:
      background-color 0.16s ease,
      border-color 0.16s ease;
    will-change: background-color, border-color;
    border-left: 0.18vh solid transparent;
  }
  .menu-item:hover {
    background-color: var(--accent-soft);
    border-left-color: var(--accent);
  }
  .menu-item.active {
    background-color: var(--accent-soft);
    border-left-color: var(--accent-600);
  }

  .text-input {
    width: 14vh;
    padding: 0.5vh 0.8vh;
    border-radius: 0.3vh;
    outline: none;
    border: 0.12vh solid rgba(255, 255, 255, 0.18);
    background: var(--bg-800);
    color: var(--text);
    font-size: 1.2vh;
    box-shadow: inset 0 0 0.25vh rgba(0, 0, 0, 0.35);
    transition:
      border-color 0.15s ease,
      box-shadow 0.15s ease;
  }
  .text-input:focus {
    border-color: var(--accent);
    box-shadow: 0 0 0 0.15vh rgba(122, 77, 255, 0.35);
  }

  .key-pill {
    padding: 0.35vh 0.7vh;
    border-radius: 0.3vh;
    background: var(--bg-800);
    color: var(--text);
    font-size: 1.1vh;
    border: 0.12vh solid rgba(255, 255, 255, 0.24);
  }
  .capture-pill {
    background: var(--accent);
    color: #fff;
  }

  .player-popout {
    position: absolute;
    left: calc(1.75vh + 33.5vh + 2vh);
    top: 10.4vh;
    width: 18.5vh;
    color: #e9dcff;
    background: rgba(12, 10, 14, 0.98);
    border: 1px solid rgba(120, 80, 160, 0.45);
    border-radius: 0.4vh;
    padding: 1.1vh 1.2vh;
    box-shadow: 0 0.6vh 1.8vh rgba(0, 0, 0, 0.4);
  }
  .pop-title {
    font-weight: 700;
    margin-bottom: 0.8vh;
    letter-spacing: 0.02em;
    font-size: 1.25vh;
  }
  .kv {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin: 0.4vh 0;
    font-size: 1vh;
  }
  .kv .health {
    color: #ff5a66;
  }
  .kv .label {
    opacity: 0.85;
  }
  .kv .value {
    font-weight: 600;
  }
  .bar {
    height: 0.6vh;
    background: rgba(255, 255, 255, 0.08);
    border-radius: 0.4vh;
    overflow: hidden;
    margin: 0.4vh 0 0.2vh;
  }
  .fill {
    height: 100%;
    background: linear-gradient(90deg, #9a5efc, #b47bff);
  }

  .openkey-pop {
    position: fixed;
    right: 18px;
    bottom: 18px;
    width: 260px;
    background: rgba(12, 10, 14, 0.98);
    color: #e9dcff;
    border: 1px solid rgba(120, 80, 160, 0.45);
    border-radius: 4px;
    padding: 12px 14px;
    box-shadow: 0 6px 18px rgba(0, 0, 0, 0.4);
    z-index: 9999;
  }
  .openkey-pop .t {
    font-weight: 700;
    margin-bottom: 6px;
  }
  .openkey-pop .v {
    margin: 4px 0;
  }
  .openkey-pop .h {
    font-size: 12px;
    opacity: 0.7;
  }
  @keyframes menuPop {
    from {
      opacity: 0;
      transform: translateY(8px) scale(0.98);
    }
    to {
      opacity: 1;
      transform: translateY(0) scale(1);
    }
  }
  .menu-root {
    will-change: transform, opacity;
    transform: translateZ(0);
  }

  .banner-title {
    text-shadow: 0 0.08vh 0.32vh rgba(0, 0, 0, 0.95);
  }

  .menu-root {
    box-shadow: 0 0.5vh 1.8vh rgba(0, 0, 0, 0.4);
    position: relative;
  }
  .banner-card {
    box-shadow: 0 0.4vh 1.6vh rgba(0, 0, 0, 0.35);
  }
  .list-shell::before {
    content: "";
    position: absolute;
    left: 0;
    right: 0;
    top: 0;
    height: 0.7vh;
    background: linear-gradient(
      to bottom,
      rgba(122, 77, 255, 0.35),
      transparent
    );
    pointer-events: none;
  }
  .menu-item {
    transition:
      background-color 0.16s ease,
      border-color 0.16s ease,
      transform 0.14s ease;
    will-change: background-color, border-color, transform;
  }
  .menu-item:nth-child(even) {
    background-color: rgba(255, 255, 255, 0.01);
  }
  .menu-item:hover {
    background-color: var(--accent-soft);
    border-left-color: var(--accent);
    transform: translateX(0.2vh);
  }
  .menu-item.active {
    background-color: var(--accent-soft);
    border-left-color: var(--accent-600);
    transform: translateX(0.3vh);
  }
  .menu-item > span:first-child {
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }
  .footer-bar {
    font-size: 1.05vh;
    letter-spacing: 0.03em;
  }
  .footer-bar span:first-child {
    text-transform: uppercase;
    opacity: 0.85;
  }
  .footer-bar span:last-child {
    padding: 0.1vh 0.5vh;
    border-radius: 0.2vh;
    background: rgba(255, 255, 255, 0.04);
    font-weight: 600;
  }
  .spectator-box {
    position: fixed;
    top: 3vh;
    right: 3vh;
    min-width: 14vh;
    max-width: 24vh;
    background: rgba(0, 0, 0, 0.92);
    color: #fff;
    border-radius: 0.5vh;
    padding: 0.9vh 0.9vh 0.8vh;
    border: 0.12vh solid rgba(255, 255, 255, 0.2);
    box-shadow: 0 0.6vh 1.8vh rgba(0, 0, 0, 0.6);
    font-size: 1vh;
    z-index: 9999;
  }
  .spec-title {
    font-weight: 700;
    letter-spacing: 0.16em;
    font-size: 1vh;
    margin-bottom: 0.5vh;
  }
  .spec-list {
    list-style: none;
    padding: 0;
    margin: 0;
  }
  .spec-count {
    font-weight: 600;
    font-size: 1.3vh;
    margin-top: 0.1vh;
  }
  .spec-empty {
    opacity: 0.8;
  }
  .toggle:active::before {
    transform: scale(0.96) translateX(1.8vh);
  }
</style>
