<script>
	// @ts-nocheck
	import Window from '$lib/utils/window/window.svelte';
	import Notifications from '$lib/utils/notifications.svelte';
	import {
		windowList,
		minimizedSig,
		activeSignal,
		focusWindowTop,
		notif
	} from '$lib/stores/index.js';
	import '$lib/style/os.css';
	import { onMount } from 'svelte';
	import mainBG from '$lib/img/bg/default.jpg';
	import browser from '$lib/img/icons/earthWhite.png';
	import g from '$lib/img/icons/controller.png';
	import a from '$lib/img/icons/apps.png';
	import s from '$lib/img/icons/settings.png';
	import y from '$lib/img/icons/swap.png';
	import p from '$lib/img/icons/pyrite.png';
	import sp from '$lib/img/icons/nvidia.png';
	import { get } from 'svelte/store';
	import { applyStartupSettings } from '$lib/utils/cloak.js';
	import { saveTheme } from '$lib/utils/theme.js';
	import gsap from 'gsap';
	import { Warp, LiquidMetal } from '@devmischief/shaders-svelte';
	import { loadSetting, saveSetting, onSettingChange } from '$lib/utils/localspace.js';
	import faviconFetch from 'favicon-fetch';
	import { launchAds } from '$lib/utils/ads';

	// Change this to rename the desktop branding (start menu).
	const siteName = 'GalaxyV7';
	let startOpen = $state(false);
	let query = $state('');
	let dateString = $state('');
	let colorMode = $state('dark');
	let activeButton = $state(null);
	let menuOpen = $state(false);
	let openMenuX = $state(0);
	let menuX = $state(0);
	let menuY = $state(0);
	let menuSender = $state(null);
	let previewOpen = $state(false);
	let previewApp = $state(null);
	let hoverTimeout = null;
	let timeString = $state(null);
	let bgURL = $state(null);
	let hydrated = $state(false);
	let navSizeMulti = $state(0);
	let temp = 0;
	let accumY = 0;
	let dragging = $state(false);
	let windowComps = [];
	let hostname = $state(null);
	let proxApi = '%2Fapi%3Furl%3D%7Burl%7D%26type%3Dprism%26transport%3DlibcurlRaw%26autoSW%3Dfalse';
	function startNavResize(e) {
		dragging = true;
		temp = navSizeMulti;
		accumY = 0;
		e.currentTarget.requestPointerLock();
		addEventListener('mousemove', dragStart);
		addEventListener('mouseup', dragStop);
		document.addEventListener('pointerlockchange', onPointerLockChange);
	}
	function dragStart(e) {
		accumY += e.movementY;
		navSizeMulti = Math.max(Math.min(temp - accumY * 0.2, 39), -15);
	}
	function dragStop(e) {
		removeEventListener('mousemove', dragStart);
		removeEventListener('mouseup', dragStop);
		document.removeEventListener('pointerlockchange', onPointerLockChange);
		if (document.pointerLockElement) document.exitPointerLock();
		dragging = false;
		callTaskbarHeight();
	}
	function onPointerLockChange() {
		if (!document.pointerLockElement) dragStop();
	}
	const baseApps = $derived([
		{
			id: 1,
			url: '/slate',
			name: 'Browser',
			icon: browser,
			height: '50%',
			width: '50%',
			top: 100,
			left: 60
		},
		{
			id: 6,
			url: '/api?url=https://play.geforcenow.com/mall&type=prism&autoSW=false',
			name: 'GeForce',
			icon: sp,
			height: '50%',
			width: '50%',
			top: 50,
			left: 210
		},
		{
			id: 2,
			url: '/books',
			name: 'Books',
			icon: g,
			height: '50%',
			width: '50%',
			top: 120,
			left: 110
		},
		{
			id: 3,
			url: `https://${hostname}/pyrite/?api=https%3A%2F%2F${hostname}${proxApi}`,
			name: 'Pyrite',
			icon: p,
			height: '50%',
			width: '50%',
			top: 30,
			left: 110
		},
		{
			id: 4,
			url: '/apps',
			name: 'Apps',
			icon: a,
			height: '50%',
			width: '50%',
			top: 90,
			left: 160
		},
		{
			id: 5,
			url: '/settings',
			name: 'Settings',
			icon: s,
			height: '80%',
			width: '80%',
			top: 0,
			left: 0,
			center: true
		}
	]);
	let customApps = $state([]);
	const apps = $derived([...baseApps, ...customApps]);
	function conertToPixies(size, viewportLength) {
		const sizeAsText = String(size);
		const isPercentage = sizeAsText.endsWith('%');

		if (isPercentage) {
			const percent = parseFloat(sizeAsText);
			const fraction = percent / 100;
			return fraction * viewportLength;
		} else {
			return parseFloat(sizeAsText);
		}
	}

	function getCenteredPosition(width, height) {
		const windowWidthInPixels = conertToPixies(width, window.innerWidth);
		const windowHeightInPixels = conertToPixies(height, window.innerHeight);

		const leftoverWidth = window.innerWidth - windowWidthInPixels;
		const leftoverHeight = window.innerHeight - windowHeightInPixels;

		const centeredLeft = leftoverWidth / 2;
		const centeredTop = leftoverHeight / 2;

		return {
			left: centeredLeft,
			top: centeredTop
		};
	}

	function getAppConfig(appId) {
		for (const app of apps) {
			if (app.id === appId) {
				return app;
			}
		}
		return null;
	}

	$effect(() => {
		activeButton = $activeSignal;
		if (!hydrated) return;
		saveSetting('navbarsize', navSizeMulti);
	});
	let navMounted = false;
	let navIndex = 0;
	function animateNavButton(node) {
		const delay = navMounted ? 0 : 0.1 + navIndex++ * 0.09;
		gsap.fromTo(
			node,
			{
				opacity: 0,
				y: 50
			},
			{
				opacity: 1,
				y: 0,
				delay
			}
		);
	}
	onMount(() => {
		applyStartupSettings();
		navMounted = true;
		hostname = location.hostname;
		if (hostname.includes('localhost')) {
			hostname = 'v7.galxy.it.com';
		}
		document.addEventListener('mousedown', launchAds);
	});

	function getAppWindows(appId) {
		const list = get(windowList);
		const matching = [];
		for (const win of list) {
			if (win.sender === appId || win.parentApp === appId) {
				matching.push(win);
			}
		}
		return matching;
		``;
	}

	function isAppActive(appId) {
		if (activeButton === appId) return true;
		const list = get(windowList);
		for (const win of list) {
			if (win.parentApp === appId && win.sender === activeButton) {
				return true;
			}
		}
		return false;
	}

	function getPreviewWindows() {
		const matching = [];
		for (const win of $windowList) {
			if (win.sender === previewApp || win.parentApp === previewApp) {
				matching.push(win);
			}
		}
		return matching;
	}

	function hasMinimizedWindow(appId) {
		for (const win of $windowList) {
			const belongsToApp = win.sender === appId || win.parentApp === appId;
			if (belongsToApp && win.minimized) {
				return true;
			}
		}
		return false;
	}
	function openWindow(url, name, height, width, top, left, appId) {
		if (localStorage.getItem('firstVisit') == 'false') {
		} else {
			localStorage.setItem('firstVisit', 'false');
			$notif = 'Right click an app to open new window!';
		}
		let appWindows = getAppWindows(appId);
		if (appWindows.length > 1) {
			previewApp = appId;
			previewOpen = true;
			return;
		}
		if (appWindows.length === 1) {
			minimizedSig.set(appWindows[0].sender);
			previewOpen = false;
			return;
		}

		activeSignal.set(appId);

		const appConfig = getAppConfig(appId);
		if (appConfig && appConfig.center) {
			const centeredPosition = getCenteredPosition(width, height);
			top = centeredPosition.top;
			left = centeredPosition.left;
		}

		const queryString = url.split('?')[1];
		if (queryString) {
			const params = new URLSearchParams(queryString);
			const notifMsg = params.get('notif');
			if (notifMsg) {
				$notif = notifMsg;
			}
		}
		windowList.update((list) => [
			...list,

			{
				url,
				name,
				height,
				width,
				top,
				left,
				id: `win-${Date.now()}`,
				sender: appId,
				parentApp: appId
			}
		]);
	}

	function openNewWindow(url, name, height, width, top, left, appId) {
		let uniqueSender = `${appId}-${Date.now()}`;
		let newName = String(name) + ' (' + getAppWindows(appId).length + ')';
		name = newName;
		activeSignal.set(uniqueSender);

		const appConfig = getAppConfig(appId);
		if (appConfig && appConfig.center) {
			const centeredPosition = getCenteredPosition(width, height);
			top = centeredPosition.top;
			left = centeredPosition.left;
		}

		windowList.update((list) => [
			...list,
			{
				url,
				name,
				height,
				width,
				top,
				left,
				id: `win-${Date.now()}`,
				sender: uniqueSender,
				parentApp: appId
			}
		]);
		closeMenu();
	}

	function focusWindow(sender) {
		focusWindowTop.set(sender);
		previewOpen = false;
	}
	let menuType = $state(null);
	function openMenu(e, type, appId, url, name, height, width, top, left) {
		e.preventDefault();
		e.stopPropagation();
		menuX = e.clientX;
		menuY = e.clientY;
		openMenuX = e.clientX;
		if (type == 'navBar') {
			menuType = 'nav';
			menuOpen = true;
		} else if (type == 'apps') {
			menuSender = { appId, url, name, height, width, top, left };
			menuType = 'app';
			hoverEnd();
			menuOpen = true;
		}
	}

	function closeMenu() {
		menuOpen = false;
		menuSender = null;
		startOpen = false;
	}

	function toggleStart(e) {
		e.stopPropagation();
		menuOpen = false;
		previewOpen = false;
		query = '';
		startOpen = !startOpen;
	}

	function launch(app) {
		startOpen = false;
		openWindow(app.url, app.name, app.height, app.width, app.top, app.left, app.id);
	}

	function readColorMode() {
		colorMode = document.documentElement.dataset.theme === 'glass-light' ? 'light' : 'dark';
	}

	async function toggleColorMode() {
		await saveTheme(colorMode === 'light' ? 'glass-dark' : 'glass-light');
		readColorMode();
	}

	const filteredApps = $derived(
		apps.filter((app) => String(app.name).toLowerCase().includes(query.trim().toLowerCase()))
	);

	function focusOnMount(node) {
		node.focus();
	}

	const proxies = [
		{ value: 'prism', label: 'SJ2' },
		{ value: 'polygon', label: 'SJ' },
		{ value: 'glass', label: 'UV' }
	];
	const transports = [
		{ value: 'libcurl', label: 'Libcurl' },
		{ value: 'epoxy', label: 'Epoxy' }
	];

	let addOpen = $state(false);
	let advancedOpen = $state(false);
	let addError = $state(null);
	let newName = $state('');
	let newUrl = $state('');
	let newIcon = $state('');
	let newProxy = $state('prism');
	let newTransport = $state('libcurl');
	let newWisp = $state('');
	let newNotif = $state('');

	function openAddApp() {
		closeMenu();
		addError = null;
		advancedOpen = false;
		newName = '';
		newUrl = '';
		newIcon = '';
		newProxy = 'prism';
		newTransport = 'libcurl';
		newWisp = '';
		newNotif = '';
		addOpen = true;
	}

	function closeAddApp() {
		addOpen = false;
	}

	function hostnameOf(url) {
		try {
			return new URL(/^https?:\/\//i.test(url) ? url : `https://${url}`).hostname;
		} catch {
			return null;
		}
	}
	function nextWindowPosition() {
		const step = customApps.length % 6;
		return { top: 60 + step * 24, left: 90 + step * 40 };
	}

	function addApp() {
		const url = newUrl.trim();
		const name = newName.trim();
		if (!url || !name) {
			addError = 'Name and URL are required';
			return;
		}

		const params = new URLSearchParams({ url, type: newProxy, transport: newTransport });
		if (newNotif.trim()) params.set('notif', newNotif.trim());
		if (newWisp.trim()) params.set('wisp', newWisp.trim());

		const host = hostnameOf(url);
		const { top, left } = nextWindowPosition();
		customApps.push({
			id: `custom-${Date.now()}`,
			url: `/api?${params.toString()}`,
			name,
			icon: newIcon.trim() || (host ? faviconFetch({ hostname: host }) : browser),
			height: '50%',
			width: '50%',
			top,
			left
		});
		saveSetting('customApps', $state.snapshot(customApps));
		addOpen = false;
	}

	function removeApp(appId) {
		customApps = customApps.filter((app) => app.id !== appId);
		saveSetting('customApps', $state.snapshot(customApps));
		closeMenu();
	}

	function isCustomApp(appId) {
		for (const app of customApps) {
			if (app.id === appId) return true;
		}
		return false;
	}

	function hoverStart(e, appId) {
		previewOpen = false;
		previewApp = null;
		e.preventDefault();
		e.stopPropagation();
		menuX = e.clientX;
		hoverTimeout = setTimeout(() => {
			let appWindows = getAppWindows(appId);
			if (appWindows.length > 0) {
				previewApp = appId;
				previewOpen = true;
			}
		}, 400);
	}
	let closeTimeout = null;
	function hoverEnd() {
		clearTimeout(hoverTimeout);
		closeTimeout = setTimeout(() => {
			previewOpen = false;
			previewApp = null;
		}, 200);
	}
	function updateTime() {
		const now = new Date();
		timeString = now.toLocaleTimeString([], { hour: 'numeric', minute: '2-digit' });
		dateString = now.toLocaleDateString([], { month: 'short', day: 'numeric' });
	}

	onMount(async () => {
		bgURL = await loadSetting('bg', mainBG);
		navSizeMulti = await loadSetting('navbarsize', 0);
		customApps = await loadSetting('customApps', []);
		hydrated = true;
		readColorMode();
		const themeWatcher = new MutationObserver(readColorMode);
		themeWatcher.observe(document.documentElement, {
			attributes: true,
			attributeFilter: ['data-theme']
		});
		updateTime();
		const interval = setInterval(updateTime, 1000);
		const unsubscribeBG = onSettingChange('bg', (value) => {
			bgURL = value ?? mainBG;
		});
		const unsubscribeApps = onSettingChange('customApps', (value) => {
			customApps = value ?? [];
		});
		return () => {
			themeWatcher.disconnect();
			clearInterval(interval);
			unsubscribeBG();
			unsubscribeApps();
		};
	});

	function switchMode() {
		if (localStorage.getItem('mode') == 'website') {
			localStorage.setItem('mode', 'os');
			location.replace('/');
		} else {
			localStorage.setItem('mode', 'website');
			location.replace('/');
		}
	}
	function callTaskbarHeight() {
		for (const comp of windowComps) {
			comp?.updateTaskbarHeight();
		}
	}
	function openChangelogs() {
		openWindow(
			'/changelog',
			'Changelogs',
			'50%',
			'600px',
			'25',
			'0',
			`win-${Date.now()}`,
			Date.now(),
			Date.now()
		);
	}
</script>
<svelte:window
	onclick={() => {
		closeMenu();
	}}
	onblur={() => (startOpen = false)}
	onkeydown={(e) => {
		if (e.key === 'Escape') {
			if (addOpen) closeAddApp();
			else startOpen = false;
		}
	}}
/>

<Notifications />

<div class="background" style="background-image: url({bgURL});"></div>
<div class="wallpaperTint"></div>

<div class="desktopIcons noSelect">
	{#each baseApps as app (app.id)}
		<button class="desktopIcon" ondblclick={() => launch(app)} onclick={(e) => e.stopPropagation()}>
			<span class="tile"><img src={app.icon} alt="" /></span>
			<span class="iconLabel">{app.name}</span>
		</button>
	{/each}
</div>

{#if menuOpen}
	<div class="contextMenu" style="left: {openMenuX}px;" onclick={(e) => e.stopPropagation()}>
		{#if menuType == 'app'}
			<button
				class="menuOption"
				onclick={() =>
					openNewWindow(
						menuSender.url,
						menuSender.name,
						menuSender.height,
						menuSender.width,
						menuSender.top,
						menuSender.left,
						menuSender.appId
					)}
			>
				Open New Window
			</button>
			{#if isCustomApp(menuSender.appId)}
				<button class="menuOption danger" onclick={() => removeApp(menuSender.appId)}>
					Remove App
				</button>
			{/if}
		{:else if menuType == 'nav'}
			<button class="menuOption" onclick={openAddApp}> Add New App </button>
		{/if}
	</div>
{/if}
{#if addOpen}
	<div class="modalOverlay" onclick={closeAddApp}>
		<div
			class="modal"
			onclick={(e) => e.stopPropagation()}
			{@attach (node) => {
				gsap.fromTo(
					node,
					{ y: 12, opacity: 0, scale: 0.97 },
					{ y: 0, opacity: 1, scale: 1, duration: 0.25, ease: 'power2.out' }
				);
			}}
		>
			<p class="modalTitle">Add App</p>

			<p class="modalLabel">URL</p>
			<input
				class="modalInput"
				type="text"
				placeholder="example.com"
				bind:value={newUrl}
				oninput={() => (addError = null)}
				onkeydown={(e) => e.key === 'Enter' && addApp()}
			/>

			<p class="modalLabel">Name</p>
			<input
				class="modalInput"
				type="text"
				placeholder="Example"
				bind:value={newName}
				oninput={() => (addError = null)}
				onkeydown={(e) => e.key === 'Enter' && addApp()}
			/>

			<p class="modalLabel">Icon</p>
			<input
				class="modalInput"
				type="text"
				placeholder="leave blank to auto generate"
				bind:value={newIcon}
				onkeydown={(e) => e.key === 'Enter' && addApp()}
			/>

			<button class="advancedToggle" onclick={() => (advancedOpen = !advancedOpen)}>
				<span class="advancedArrow" class:open={advancedOpen}></span> Advanced
			</button>

			{#if advancedOpen}
				<div
					class="advancedPanel"
					{@attach (node) => {
						gsap.fromTo(node, { opacity: 0, y: -6 }, { opacity: 1, y: 0, duration: 0.2 });
					}}
				>
					<p class="modalLabel">Proxy</p>
					<div class="modalOptions">
						{#each proxies as proxy}
							<button
								class="modalOption"
								class:active={newProxy === proxy.value}
								onclick={() => (newProxy = proxy.value)}>{proxy.label}</button
							>
						{/each}
					</div>

					<p class="modalLabel">Transport</p>
					<div class="modalOptions">
						{#each transports as transport}
							<button
								class="modalOption"
								class:active={newTransport === transport.value}
								onclick={() => (newTransport = transport.value)}>{transport.label}</button
							>
						{/each}
					</div>

					<p class="modalLabel">Wisp</p>
					<input
						class="modalInput"
						type="text"
						placeholder="keep blank for default"
						bind:value={newWisp}
					/>

					<p class="modalLabel">Open Notification</p>
					<input
						class="modalInput"
						type="text"
						placeholder="shown when the app opens"
						bind:value={newNotif}
					/>
				</div>
			{/if}

			{#if addError}
				<p class="modalError">{addError}</p>
			{/if}

			<div class="modalActions">
				<button class="modalBtn" onclick={closeAddApp}>Cancel</button>
				<button class="modalBtn primary" onclick={addApp}>Add</button>
			</div>
		</div>
	</div>
{/if}
{#if previewOpen}
	<div
		style="left:{menuX}px"
		id="previewPanel"
		class="previewPanel"
		onclick={(e) => e.stopPropagation()}
		{@attach (node) => {
			gsap.fromTo(
				node,
				{ y: 20, opacity: 0 },
				{
					y: 0,
					opacity: 1,
					duration: 0.3,
					ease: 'power2.out'
				}
			);
		}}
		onmouseenter={() => clearTimeout(closeTimeout)}
		onmouseleave={() => {
			previewOpen = false;
			previewApp = null;
		}}
	>
		{#each getPreviewWindows() as win}
			<button class="previewCard" onclick={() => focusWindow(win.sender)}>
				<p>{win.name}</p>
			</button>
		{/each}
	</div>
{/if}

{#if startOpen}
	<div
		class="startMenu"
		role="dialog"
		aria-label="Start menu"
		tabindex="-1"
		onclick={(e) => e.stopPropagation()}
		{@attach (node) => {
			gsap.fromTo(
				node,
				{ y: 24, opacity: 0, scale: 0.98 },
				{ y: 0, opacity: 1, scale: 1, duration: 0.22, ease: 'power3.out' }
			);
		}}
	>
		<div class="searchBox">
			<svg viewBox="0 0 24 24" width="16" height="16" aria-hidden="true">
				<circle cx="11" cy="11" r="7" fill="none" stroke="currentColor" stroke-width="2" />
				<path d="M20 20l-4-4" stroke="currentColor" stroke-width="2" stroke-linecap="round" />
			</svg>
			<input
				type="text"
				placeholder="Search apps"
				bind:value={query}
				{@attach focusOnMount}
				onkeydown={(e) => {
					if (e.key === 'Enter' && filteredApps.length > 0) launch(filteredApps[0]);
				}}
			/>
		</div>

		<p class="startLabel">{query ? 'Results' : 'Apps'}</p>
		<div class="startGrid">
			{#each filteredApps as app (app.id)}
				<button
					class="startApp"
					onclick={() => launch(app)}
					oncontextmenu={(e) =>
						openMenu(e, 'apps', app.id, app.url, app.name, app.height, app.width, app.top, app.left)}
				>
					<span class="tile"><img src={app.icon} alt="" /></span>
					<span class="startAppName">{app.name}</span>
				</button>
			{:else}
				<p class="emptyState">No apps match “{query}”</p>
			{/each}
		</div>

		<div class="startFooter">
			<button class="footerBrand" onclick={openChangelogs} title="Changelog">
				<span class="brandDot"></span>
				{siteName}
			</button>
			<div class="footerActions">
				<button class="footerBtn" onclick={openAddApp}>+ Add app</button>
				<button class="footerBtn" onclick={switchMode}>Switch mode</button>
			</div>
		</div>
	</div>
{/if}

<div class="nav" style="--nav-h: {48 + navSizeMulti}px;">
	<div class="navResize" onmousedown={startNavResize}></div>
	<div class="navGrid noSelect" oncontextmenu={(e) => openMenu(e, 'navBar')}>
		<div class="navSide"></div>

		<div class="navStuff">
			<button
				class="startButton"
				class:open={startOpen}
				onclick={toggleStart}
				aria-label="Start"
				aria-expanded={startOpen}
			>
				<svg viewBox="0 0 24 24" width="22" height="22" aria-hidden="true">
					<rect x="3" y="3" width="8" height="8" rx="2" fill="currentColor" />
					<rect x="13" y="3" width="8" height="8" rx="2" fill="currentColor" opacity="0.75" />
					<rect x="3" y="13" width="8" height="8" rx="2" fill="currentColor" opacity="0.75" />
					<rect x="13" y="13" width="8" height="8" rx="2" fill="currentColor" opacity="0.5" />
				</svg>
			</button>
			<span class="navDivider"></span>
			{#each apps as app (app.id)}
				<button
					class="navButton"
					{@attach animateNavButton}
					class:active={isAppActive(app.id)}
					class:hasMinimized={hasMinimizedWindow(app.id) || getAppWindows(app.id).length > 0}
					title={app.name}
					onclick={() =>
						openWindow(app.url, app.name, app.height, app.width, app.top, app.left, app.id)}
					oncontextmenu={(e) =>
						openMenu(e, 'apps', app.id, app.url, app.name, app.height, app.width, app.top, app.left)}
					onmouseenter={(e) => hoverStart(e, app.id)}
					onmouseleave={hoverEnd}
				>
					<img class="navIcon" src={app.icon} alt={app.name} />
				</button>
			{/each}
		</div>

		<div class="navSide tray">
			<button
				class="trayButton"
				onclick={(e) => {
					e.stopPropagation();
					toggleColorMode();
				}}
				aria-label="Toggle light and dark mode"
				title={colorMode === 'light' ? 'Switch to dark' : 'Switch to light'}
			>
				{#if colorMode === 'light'}
					<svg viewBox="0 0 24 24" width="18" height="18" aria-hidden="true">
						<path
							d="M20 14.5A8 8 0 019.5 4a8 8 0 1010.5 10.5z"
							fill="none"
							stroke="currentColor"
							stroke-width="2"
							stroke-linejoin="round"
						/>
					</svg>
				{:else}
					<svg viewBox="0 0 24 24" width="18" height="18" aria-hidden="true">
						<circle cx="12" cy="12" r="4" fill="none" stroke="currentColor" stroke-width="2" />
						<path
							d="M12 2v2M12 20v2M2 12h2M20 12h2M5 5l1.5 1.5M17.5 17.5L19 19M5 19l1.5-1.5M17.5 6.5L19 5"
							stroke="currentColor"
							stroke-width="2"
							stroke-linecap="round"
						/>
					</svg>
				{/if}
			</button>
			<button class="trayButton" onclick={switchMode} aria-label="Switch mode" title="Switch mode">
				<img src={y} alt="" />
			</button>
			<div class="clock">
				<span class="clockTime">{timeString}</span>
				<span class="clockDate">{dateString}</span>
			</div>
		</div>
	</div>
</div>

{#each $windowList as window, i (window.id)}
	<Window
		bind:this={windowComps[i]}
		url={window.url}
		name={window.name}
		height={window.height}
		width={window.width}
		top={window.top}
		left={window.left}
		id={window.id}
		sender={window.sender}
	/>
{/each}
