<script>
    /**
     * Main 3Demos control panel, to the left of the scene.
     */
    import { onMount } from 'svelte';
    import { slide } from 'svelte/transition';
    import { quintOut } from 'svelte/easing';
    // import { flip } from 'svelte/animate';

    import { TabContent, TabPane } from '@sveltestrap/sveltestrap';

    import HowTo from './docs/HowTo.svelte';
    import About from './docs/About.svelte';
    import Polls from './polls/Polls.svelte';

    import {
        // makeHSLColor,
        querySelectorIncludesText,
        tripleToHex,
        joinUrl,
        processSearchEncoding,
        filterBang,
    } from './utils';
    import { publishScene } from './sceneUtils';

    import Session from './session/Session.svelte';

    import Surface from './objects/Surface.svelte';
    import Level from './objects/Level.svelte';
    import Curve from './objects/Curve.svelte';
    import Field from './objects/Field.svelte';
    import Function from './objects/Function.svelte';
    import Vector from './objects/Vector.svelte';
    import Point from './objects/Point.svelte';
    import Solid from './objects/Solid.svelte';

    // Can hold the component type
    let SceneObjectKind;

    import { evaluate_cmap } from './js-colormaps';
    import { colorMap } from './stores';
    import { demoObjects } from './states.svelte';
    import Story from './Story.svelte';
    import { tick } from 'svelte';
    // import { add } from 'mathjs';

    let {
        isMobileView,
        debug,
        currentMode,
        currentControls,
        setTarget,
        gridStep,
        gridMax,
        isHost,
        onRenderObject,
        onDestroyObject,
        socket,
        pollResponses,
        animateIfNotAnimating,
        isPollsOpen,
        roomId,
        currentPoll,
        currentCamera,
        scene,
        requestFrameIfNotRequested,
        blowUpObjects,
        selectedObjects,
        selectObject,
        chatBuffer,
        lockPoll,
        objectResponses,
        showPanel = $bindable(true),
    } = $props();

    let selectedPoint = $state();

    const kindToComponent = {
        point: Point,
        vector: Vector,
        field: Field,
        graph: Function,
        curve: Curve,
        level: Level,
        surface: Surface,
        solid: Solid,
    };

    const onPublishScene = function () {
        publishScene(demoObjects, selectedObjects, socket);
    };

    let nextHue = 0;
    const nextColorUp = () => {
        const cm = evaluate_cmap(nextHue, $colorMap);
        nextHue += 1 / Math.sqrt(3);
        nextHue = nextHue % 1;
        // console.log('colorerer', nextHue, $colorMap);
        return tripleToHex(cm);
    };

    /**
     * This is a object of functions, keyed on the 3demos object kind, which return a (randomized) default
     * for each object kind.
     */
    const defaultParams = {
        point: () => ({
            a: `${Math.round(10 * (2 * Math.random() - 1)) / 10}`,
            b: `${Math.round(10 * (2 * Math.random() - 1)) / 10}`,
            c: `${Math.round(10 * (2 * Math.random() - 1)) / 10}`,
            t0: '0',
            t1: '1',
        }),
        vector: () => ({
            a: `${Math.round(10 * (2 * Math.random() - 1)) / 10}`,
            b: `${Math.round(10 * (2 * Math.random() - 1)) / 10}`,
            c: `${Math.round(10 * (2 * Math.random() - 1)) / 10}`,
            x: '0',
            y: '0',
            z: '0',
            t0: '0',
            t1: '1',
        }),
        curve: () => ({
            a: '0',
            b: '2*pi',
            x: 'cos(t)',
            y: 'sin(t)',
            z: `${
                1 / 4 + Math.round(10 * Math.random()) / 10
            } * cos(${Math.ceil(10 * Math.random()).toString()}*t)`,
        }),
        graph: () => ({
            a: '-2',
            b: '2',
            c: '-2',
            d: '2',
            z: `${Math.ceil(
                4 * Math.random(),
            ).toString()} / 4 * cos(${Math.ceil(
                3 * Math.random(),
            ).toString()}*x + ${Math.ceil(
                2 * Math.random(),
            ).toString()}*y)/(1 + x^2 + y^2)`,
            t0: '0',
            t1: '1',
        }),
        level: () => ({
            g: (Math.random() > 0.5 ? '-' : '') + 'x^2 + 2 y^2 - z^2',
            k: (Math.ceil(16 * Math.random()) / 2 - 4).toString(),
            a: '-2',
            b: '2',
            c: '-2',
            d: '2',
            e: '-2',
            f: '2',
        }),
        surface: () => {
            const k = Math.ceil(10 * Math.random());
            const l = Math.ceil(10 * Math.random());
            const R = (k / 20).toString();
            return {
                a: '0',
                b: `${((21 - k) / 10).toString()} * pi`,
                c: `${(l / 10).toString()} * pi`,
                d: `${(l / 10 + 1).toString()} * pi`,
                x: `cos(u)*(1 + ${R}*sin(v))`,
                y: `sin(u)*(1 + ${R}*sin(v))`,
                z: `-${R}*cos(v)`,
            };
        },
        solid: () => {
            const k = Math.ceil(10 * Math.random());
            const l = Math.ceil(10 * Math.random());
            const m = Math.ceil(10 * Math.random());
            const n = Math.ceil(10 * Math.random());
            return {
                coords: 'rect',
                a: `-${k / 5}`,
                b: `${l / 5}`,
                c: `-${m / 5}`,
                d: `${n / 5}`,
                e: '0',
                f: `1 - ((x - ${m / 10})^2 + (y + ${k / 10})^2) / 8`,
            };
        },
        field: () => {
            const comps = ['1', '-1', 'x', 'y', 'z', '-x', '-y', '-z'];
            const p = comps[Math.floor(comps.length * Math.random())];
            const q = comps[Math.floor(comps.length * Math.random())];
            const r = comps[Math.floor(comps.length * Math.random())];

            return { p, q, r, nVec: '6' };
        },
    };

    let kindToAdd = $state(null);

    let selectedMainTabIndex = $state(0); // Only used in mobile view where tabs are used instead of accordion

    const randomID = function () {
        return Math.random().toString(36).substring(2, 9);
    };

    const addNewObject = function (kind) {
        const uuid = randomID(); // crypto.randomUUID(); caused issues on mobile devices

        demoObjects.push({
            uuid,
            kind: kind,
            params: defaultParams[kind](),
            color: nextColorUp(),
            animation: false,
        });

        selectedObjects = [];

        setTimeout(async () => {
            await tick();
            selectObject(uuid);
        }, 350); // why 350? I don't know. Autoscroll has some race condition I don't get.
        kindToAdd = null;
    };

    /**
     * Show the "Info" accordion item.
     */
    export const showMainPanelItem = function () {
        // Expand panel if it's hidden.
        showPanel = true;

        // Open the first accordion item.
        document.querySelector('#collapseOne').collapse('show');

        currentMode = 'session';
        // Because sveltestrap can't activate this tab programmatically:
        // https://github.com/bestguy/sveltestrap/issues/532
        const tabEl = querySelectorIncludesText(
            '.chapterBox .nav-tabs a',
            'Session',
        );
        tabEl.click();
    };

    onMount(() => {
        const urlParams = new URLSearchParams(
            processSearchEncoding(location.search),
        );
        if (urlParams.keys()) {
            // const objectHolder = {};
            urlParams.forEach((val, key) => {
                // This is bad and stupid, and hopefully it will be done better.
                // make a viewStatus object, maybe?
                if (key === 'showPanel') {
                    showPanel = !(val === 'false');
                } else if (key === 'debug') {
                    debug = val === 'true';
                    console.log('debuggery: ', debug);
                }
            });
        }
    });

    const onKeyDown = (e) => {
        if (e.target.matches('input, textarea')) {
            return;
        } else if (e.key === 'h') {
            showPanel = !showPanel;
        } else if (e.key === 'c' && selectedPoint) {
            setTarget(selectedPoint);
        }
    };
    window.addEventListener('keydown', onKeyDown, false);
</script>

<aside
    class="panel-wrapper"
    class:mobile={isMobileView}
    class:collapsed={!showPanel}
>
    <div class="demos-panel">
        <div id="panelAccordion" class:accordion={!isMobileView}>
            <!-- 3Demos logo on Panel only in desktop view -->
            {#if !isMobileView}
                <a href="/" title="Home" class="demos-logo">
                    <img
                        alt="3Demos logo"
                        src={joinUrl(window.STATIC_PREFIX, './3demos-logo.svg')}
                    />
                </a>
            {/if}

            <!-- Panel tabs only in mobile view -->
            {#if isMobileView}
                <ul id="panelTabs" class="nav nav-tabs">
                    <li class="nav-item">
                        <button
                            class="nav-link"
                            class:active={selectedMainTabIndex == 0}
                            aria-current="page"
                            onclick={() => {
                                selectedMainTabIndex = 0;
                            }}>Info</button
                        >
                    </li>
                    <li class="nav-item">
                        <button
                            class="nav-link disabled"
                            class:active={selectedMainTabIndex == 1}
                            tabindex="-1"
                            aria-disabled="true">Polls</button
                        >
                    </li>
                    <li class="nav-item">
                        <button
                            class="nav-link"
                            class:active={selectedMainTabIndex == 2}
                            onclick={() => {
                                selectedMainTabIndex = 2;
                            }}>Objects</button
                        >
                    </li>

                    <!-- Spacer -->
                    <div style="flex-grow: 1;"></div>

                    <!-- Mobile "hide panel" button -->
                    <li>
                        <button
                            class="nav-link"
                            onclick={() => (showPanel = !showPanel)}
                            title={showPanel ? 'Hide panel' : 'Show panel'}
                        >
                            {#if showPanel}
                                <i class="bi bi-arrow-bar-down"></i>
                            {:else}
                                <i class="bi bi-arrow-bar-up"></i>
                            {/if}
                        </button>
                    </li>
                </ul>
            {/if}
            <div class="demos-panel-box" class:accordion-item={!isMobileView}>
                {#if !isMobileView}
                    <h2 class="accordion-header">
                        <button
                            class="accordion-button collapsed"
                            id="info"
                            type="button"
                            data-bs-toggle="collapse"
                            data-bs-target="#collapseOne"
                            aria-expanded="false"
                            aria-controls="collapseOne"
                            onclick={() => {
                                selectedMainTabIndex = 0;
                            }}
                        >
                            Info
                        </button>
                    </h2>
                {/if}

                <div
                    id="collapseOne"
                    class:accordion-collapse={!isMobileView &&
                        selectedMainTabIndex == 0}
                    class={selectedMainTabIndex == 0 ? 'show' : 'collapse'}
                    data-bs-parent="#panelAccordion"
                >
                    <div class="chapterBox">
                        <div class="collapse-info">
                            <TabContent
                                on:tab={(e) => (currentMode = e.detail)}
                            >
                                <TabPane
                                    tabId="how-to"
                                    tab="Creation"
                                    active={currentMode === 'how-to'}
                                >
                                    <HowTo />
                                </TabPane>
                                <TabPane
                                    tabId="session"
                                    tab="Session"
                                    active={currentMode === 'session'}
                                >
                                    <Session
                                        bind:roomId
                                        bind:socket
                                        bind:currentPoll
                                        bind:chatBuffer
                                        {pollResponses}
                                        {selectedPoint}
                                        {selectedObjects}
                                        {isHost}
                                    />
                                </TabPane>
                                <TabPane
                                    tabId="story"
                                    tab="Story"
                                    active={currentMode === 'story'}
                                >
                                    <Story
                                        {scene}
                                        render={requestFrameIfNotRequested}
                                        animate={animateIfNotAnimating}
                                    />
                                </TabPane>
                                <TabPane
                                    tabId="about"
                                    tab="About"
                                    active={currentMode === 'about'}
                                >
                                    <About />
                                </TabPane>
                            </TabContent>
                        </div>
                    </div>
                </div>
            </div>
            <!-- end .demos-panel-box -->

            {#if currentMode === 'session' && isHost}
                <div
                    class="demos-panel-box"
                    class:accordion-item={!isMobileView}
                >
                    {#if !isMobileView}
                        <h2 class="accordion-header">
                            <button
                                class="accordion-button collapsed"
                                id="polls"
                                type="button"
                                data-bs-toggle="collapse"
                                data-bs-target="#collapseTwo"
                                aria-expanded="false"
                                aria-controls="collapseTwo"
                                onclick={() => {
                                    selectedMainTabIndex = 1;
                                }}
                            >
                                Polls
                            </button>
                        </h2>
                    {/if}
                    <div
                        id="collapseTwo"
                        class:accordion-collapse={!isMobileView &&
                            selectedMainTabIndex == 0}
                        class={selectedMainTabIndex == 1 ? 'show' : 'collapse'}
                        data-bs-parent="#panelAccordion"
                    >
                        <Polls
                            bind:pollResponses
                            bind:isPollsOpen
                            bind:lockPoll
                            bind:currentPoll
                            {socket}
                            {objectResponses}
                            render={requestFrameIfNotRequested}
                        />
                    </div>
                </div>
            {/if}

            <div class="demos-panel-box" class:accordion-item={!isMobileView}>
                {#if !isMobileView}
                    <h2 class="accordion-header">
                        <button
                            class="accordion-button collapsed"
                            id="objects"
                            type="button"
                            data-bs-toggle="collapse"
                            data-bs-target="#collapseThree"
                            aria-expanded="false"
                            aria-controls="collapseThree"
                            onclick={() => {
                                selectedMainTabIndex = 2;
                            }}
                        >
                            Objects
                        </button>
                    </h2>
                {/if}

                <div
                    id="collapseThree"
                    class:accordion-collapse={!isMobileView &&
                        selectedMainTabIndex == 2}
                    class={selectedMainTabIndex == 2 ? 'show' : 'collapse'}
                    data-bs-parent="#panelAccordion"
                >
                    <div class="objectBoxOuter">
                        <div class="collapse-info">
                            <div class="object-box-title d-flex mb-2">
                                <h2 class="flex-grow-1 px-2">3D Objects</h2>
                            </div>
                            <div
                                class="btn-toolbar justify-content-between"
                                role="toolbar"
                            >
                                <div class="btn-group mb-2">
                                    <select
                                        bind:value={kindToAdd}
                                        onchange={(e) => {
                                            if (e.target.value) {
                                                addNewObject(e.target.value);
                                                document.activeElement.blur();
                                            }
                                        }}
                                        class="demos-obj-select form-select bg-primary border-primary text-light"
                                    >
                                        <option value={null}>
                                            Add Object &#xFF0B;
                                        </option>
                                        <option value="point">point</option>
                                        <option value="vector">vector</option>
                                        <option value="curve">curve</option>
                                        <option value="graph">graph</option>
                                        <option value="level"
                                            >level surface</option
                                        >
                                        <option value="surface"
                                            >parametric surface</option
                                        >
                                        <option value="solid"
                                            >solid region</option
                                        >
                                        <option value="field"
                                            >vector field</option
                                        >
                                    </select>

                                    <button
                                        class="btn btn-sm btn-danger"
                                        onclick={blowUpObjects}
                                    >
                                        Clear Objects
                                        <i class="fa fa-trash"></i>
                                    </button>
                                    {#if currentMode === 'session' && isHost}
                                        <button
                                            class="btn btn-sm btn-primary"
                                            onclick={onPublishScene}
                                        >
                                            Publish Scene
                                            <i class="bi bi-broadcast-pin"></i>
                                        </button>
                                    {/if}
                                </div>

                                <div class="objectBoxInner">
                                    <!-- Main Loop, if you will -->
                                    {#each demoObjects as dobj (dobj.uuid)}
                                        {@const KindComp =
                                            kindToComponent[dobj.kind]}
                                        <!-- 
                                        <div
                                            transition:slide={{
                                                delay: 0,
                                                duration: 200,
                                                easing: quintOut,
                                            }}
                                        > -->
                                        <div>
                                            <KindComp
                                                {scene}
                                                {onRenderObject}
                                                {onDestroyObject}
                                                camera={currentCamera}
                                                controls={currentControls}
                                                render={requestFrameIfNotRequested}
                                                onClose={() => {
                                                    filterBang(
                                                        (x) =>
                                                            x.uuid !==
                                                            dobj.uuid,
                                                        demoObjects,
                                                    );
                                                }}
                                                bind:params={dobj.params}
                                                bind:color={dobj.color}
                                                bind:title={dobj.title}
                                                bind:animation={dobj.animation}
                                                meta={dobj.meta}
                                                uuid={dobj.uuid}
                                                selected={dobj.selected}
                                                {gridStep}
                                                {gridMax}
                                                animate={animateIfNotAnimating}
                                                {selectObject}
                                                selectPoint={(point) =>
                                                    (selectedPoint = point)}
                                            />
                                        </div>
                                    {/each}
                                </div>

                                <!-- debug buttons -->

                                {#if debug}
                                    <div>
                                        <button
                                            onclick={() => {
                                                demoObjects.length = 0;
                                                demoObjects.push({
                                                    uuid: 34,
                                                    kind: 'curve',
                                                    params: {
                                                        a: '0',
                                                        b: '2*pi',
                                                        x: 'cos(t)',
                                                        y: 'sin(t)',
                                                        z: '0',
                                                    },
                                                    color: '#aa33ff',
                                                    animation: true,
                                                });
                                                demoObjects.push({
                                                    uuid: '34point22',
                                                    kind: 'point',
                                                    params: {
                                                        a: 'cos(2 t)',
                                                        b: 'sin(2 t)',
                                                        c: 'cos(2 t) + sin(2 t)',
                                                        t0: '0',
                                                        t1: '2 pi',
                                                    },
                                                    color: '#FF0000',
                                                    animation: false,
                                                });
                                                demoObjects.push({
                                                    uuid: 345,
                                                    kind: 'vector',
                                                    params: {
                                                        a: 'cos(t)',
                                                        b: 'sin(t)',
                                                        c: '1',
                                                        x: 'cos(t)',
                                                        y: 'sin(t)',
                                                        z: '0',
                                                        t0: '0',
                                                        t1: '2*pi',
                                                    },
                                                    color: '#ff33ff',
                                                    animation: false,
                                                });
                                            }}>curve anim</button
                                        >
                                        <button
                                            onclick={() => {
                                                demoObjects.length = 0;
                                                demoObjects.push({
                                                    uuid: 34,
                                                    kind: 'curve',
                                                    params: {
                                                        a: '0',
                                                        b: '2*pi',
                                                        x: 'cos(t)',
                                                        y: 'sin(t)',
                                                        z: '0',
                                                    },
                                                    color: '#aa33ff',
                                                    animation: false,
                                                });
                                                demoObjects.push({
                                                    uuid: '34point22',
                                                    kind: 'point',
                                                    params: {
                                                        a: 'cos(2 t)',
                                                        b: 'sin(2 t)',
                                                        c: 'cos(2 t) + sin(2 t)',
                                                        t0: '0',
                                                        t1: '2*pi',
                                                    },
                                                    color: '#FF0000',
                                                    animation: true,
                                                });
                                                demoObjects.push({
                                                    uuid: 345,
                                                    kind: 'vector',
                                                    params: {
                                                        a: 'cos(t)',
                                                        b: 'sin(t)',
                                                        c: '1',
                                                        x: 'cos(t)',
                                                        y: 'sin(t)',
                                                        z: '0',
                                                        t0: '0',
                                                        t1: '2*pi',
                                                    },
                                                    color: '#ff33ff',
                                                    animation: true,
                                                });
                                            }}>point/vec anim</button
                                        >
                                        <button
                                            onclick={() => {
                                                demoObjects.length = 0;
                                                demoObjects.push({
                                                    uuid: 'agraph3847',
                                                    kind: 'graph',
                                                    params: {
                                                        a: '-1',
                                                        b: '1',
                                                        c: '-1',
                                                        d: '1',
                                                        z: 'x^2 - 3*cos(t) * x * y + y^2',
                                                        t0: '0',
                                                        t1: '2*pi',
                                                    },
                                                    color: '#ff33ff',
                                                    animation: true,
                                                });
                                            }}>anim func</button
                                        >
                                        <button
                                            onclick={() => {
                                                demoObjects.length = 0;
                                                demoObjects.push({
                                                    uuid: 'agraph3847',
                                                    kind: 'graph',
                                                    params: {
                                                        a: '-1',
                                                        b: '1',
                                                        c: '-1',
                                                        d: '1',
                                                        z: 'x^2 - 3*cos(t) * x * y + y^2',
                                                        t0: '0',
                                                        t1: '2*pi',
                                                    },
                                                    color: '#ff33ff',
                                                    animation: false,
                                                });
                                            }}>unanim func</button
                                        >
                                        <button
                                            onclick={() => {
                                                demoObjects.length = 0;
                                                demoObjects.push({
                                                    uuid: 'swirly1234',
                                                    kind: 'field',
                                                    params: {
                                                        p: 'x/2',
                                                        q: '-z',
                                                        r: 'y',
                                                        nVec: '5',
                                                    },
                                                    color: '#ff33ff',
                                                    animation: true,
                                                });
                                            }}>anim field</button
                                        >
                                        <button
                                            onclick={() => {
                                                demoObjects.length = 0;
                                                demoObjects.push({
                                                    uuid: 'swirly1234',
                                                    kind: 'field',
                                                    params: {
                                                        p: 'x/2',
                                                        q: '-z',
                                                        r: 'y',
                                                        nVec: '5',
                                                    },
                                                    color: '#ff33ff',
                                                    animation: false,
                                                });
                                            }}>unanim field</button
                                        >
                                    </div>
                                {/if}
                            </div>
                        </div>
                        <!-- end .objectBoxOuter -->
                    </div>
                </div>
            </div>
            <!-- end .accordion -->
        </div>
        {#if !isMobileView}
            <button
                class="panel-hider"
                onclick={() => (showPanel = !showPanel)}
                title={showPanel ? 'Hide panel' : 'Show panel'}
            >
                {#if showPanel}
                    <i class="bi bi-arrow-bar-left"></i>
                {:else}
                    <i class="bi bi-arrow-bar-right"></i>
                {/if}
            </button>
        {/if}
    </div>
</aside>

<!-- end .demos-panel -->

<!-- 
<div
    class="panel-button panel-hider bg-info bg-opacity-25 border border-info border-start-0 rounded-end-circle"
    title={showPanel ? 'Hide panel' : 'Show panel'}
    onclick={() => (showPanel = !showPanel)}
    onkeypress={() => (showPanel = !showPanel)}
    class:collapsed={!showPanel}
    style="--collapsed-width: translateX(-{panelWidth}px);"
>
    <div class="align-middle text-center">
        {#if showPanel}
            <i class="bi bi-arrow-bar-left" />
        {:else}
            <i class="bi bi-arrow-bar-right" />
        {/if}
    </div>
</div> -->

<style>
    :global(#panelAccordion .collapse-info .nav-link:not(.active)) {
        color: white;
    }

    :global(#panelTabs) {
        background-color: #efefff;
        padding-top: 5px;
    }

    .demos-logo img {
        height: 38px;
    }

    .panel-wrapper {
        position: relative;
        margin: 0;
        width: fit-content;
        height: fit-content;
        transition: transform 100ms ease-in-out;
        z-index: 1;
    }

    .demos-panel {
        min-width: 300px;
        /* default width */
        width: 370px;
        max-width: 60vw;
        height: fit-content;
        max-height: 100%;

        background-color: transparent;

        overflow-y: auto;
        overflow-x: hidden;

        resize: horizontal;
    }

    .mobile .demos-panel {
        position: fixed;
        bottom: 0;
        max-width: 100%;
        width: 100%;
        overflow-y: hidden;
    }

    .mobile.collapsed .demos-panel-box {
        display: none;
    }

    /* .panel-button {
        cursor: pointer;
        position: relative;

        min-width: 28px;
        height: 38px;

        z-index: 2;
    } */

    .panel-wrapper:not(.mobile).collapsed {
        transform: translateX(-100%);
    }

    .panel-hider {
        --width: 25px;
        width: var(--width);
        position: absolute;
        right: calc(-1 * var(--width));
        top: 46px;
        border-radius: 0 5px 5px 0;
    }

    .chapterBox,
    .objectBoxOuter {
        background-color: rgba(0, 0, 0, 0.5);
        border-radius: 0.5em;
        border-top-left-radius: 0rem;
        border-top-right-radius: 0rem;
        border: 1px solid black;
        padding: 5px;
        transition: opacity 0.8s;
        box-sizing: border-box;

        -webkit-backface-visibility: hidden;
        backface-visibility: hidden;
    }

    .mobile .chapterBox,
    .mobile .objectBoxOuter {
        height: 300px;
        overflow-y: scroll;
    }

    .chapterBox {
        text-align: unset;
    }

    .objectBoxInner {
        display: flex;
        flex-direction: column;
        gap: 0.25em;
        max-height: 60vh;
        overflow-y: auto;
        width: 100%;
    }

    .mobile .objectBoxInner {
        overflow-y: visible; /* we don't want the objectboxinner div to be scrollable on mobile as its parent is scrollable */
    }

    .object-box-title {
        display: flex;
        font-size: 1.5em;
        justify-content: space-between;
    }

    .accordion-header {
        font-family:
            system-ui,
            -apple-system,
            BlinkMacSystemFont,
            'Segoe UI',
            Roboto,
            Oxygen,
            Ubuntu,
            Cantarell,
            'Open Sans',
            'Helvetica Neue',
            sans-serif;
    }

    .demos-obj-select {
        border-bottom-right-radius: 0;
        border-top-right-radius: 0;
        width: inherit;
    }
</style>
