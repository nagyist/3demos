<script module>
    let titleIndex = 0;
</script>

<script>
    import { onMount, onDestroy, untrack } from 'svelte';
    import * as THREE from 'three';
    import { create, all } from 'mathjs';

    // import { dependsOn } from './Vector.svelte';
    import M from '../M.svelte';
    import ObjHeader from './ObjHeader.svelte';
    // import PlayButtons from '../form-components/PlayButtons.svelte';
    import { vMin, vMax, densityColormap } from '../stores';

    const config = {};
    const math = create(all, config);

    import {
        colorBufferVertices,
        vMaxMin,
        blueUpRedDown,
        // checksum,
        RectangularSolidGeometry,
        CylindricalSolidGeometry,
        SphericalSolidGeometry,
    } from '../utils.js';

    import { flashDance } from '../sceneUtils';
    import {
        plainVertexShader,
        heatmapFragmentShader,
        plainRedFrag,
    } from './shaders';

    import InputChecker from '../form-components/InputChecker.svelte';
    import ColorBar from '../settings/ColorBar.svelte';
    import Nametag from './Nametag.svelte';
    import { uniformColorData } from '../js-colormaps';
    // import ObjectParamInput from '../form-components/ObjectParamInput.svelte';

    let {
        uuid,
        onRenderObject,
        onDestroyObject,
        selectObject,
        params = $bindable(),
        color = $bindable('#5432ff'),
        title = $bindable(),
        scene,
        render,
        onClose,
        selected,
    } = $props();

    // default density
    params.mu = params.mu ?? '1';

    onMount(() => {
        titleIndex++;
        title = title || `Solid Region ${titleIndex}`;

        if (params.mu && params.mu != '1') chooseDensity = true;
    });

    onDestroy(() => {
        scene.remove(solidGroup);
        box.geometry.dispose();
        box.material.dispose();
        borders.geometry.dispose();
        borders.material.dispose();

        onDestroyObject(box);
        window.removeEventListener('keydown', onKeyDown, false);
        render();
    });

    let objHidden = $state(false);

    $effect(() => {
        solidGroup.visible = !objHidden;
    });

    const toggleHide = function () {
        objHidden = !objHidden;
        render();
    };

    const onKeyDown = (e) => {
        if (e.target.matches('input, textarea')) {
            return;
        }
        if (selected) {
            switch (e.key) {
                case 'Backspace':
                    toggleHide();
                    break;
                case 'd':
                    chooseDensity = !chooseDensity;
                    break;
            }
        }
    };

    window.addEventListener('keydown', onKeyDown, false);

    let minimize = $state(false);

    let nX = $state(60);

    let chooseDensity = $state(false);

    // export let myId;

    const colorMaterial = new THREE.MeshPhongMaterial({
        color: 0xffffff,
        shininess: 70,
        side: THREE.DoubleSide,
        vertexColors: true,
        transparent: false,
    });

    const colorMeBadd = (mesh, f) => {
        const [vMaxLocal, vMinLocal] = vMaxMin(mesh, f);
        $vMin = Math.min($vMin, vMinLocal);
        $vMax = Math.max($vMax, vMaxLocal);
        if ($vMax == $vMin) {
            if ($vMax == 0) {
                $vMax = 1;
                $vMin = -1;
            } else {
                $vMax = (4 / 3) * Math.abs($vMax);
                $vMin = (-4 / 3) * Math.abs($vMin);
            }
        }

        colorBufferVertices(mesh, (x, y, z) => {
            const value = f(x, y, z);
            return blueUpRedDown(
                (2 * (value - $vMin)) / ($vMax - $vMin) - 1,
                0.8,
                $densityColormap,
            );
        });
    };

    $effect(() => {
        console.log('shader effex');
        if (chooseDensity && $densityColormap) {
            // params.mu = params.mu || '1';
            // compiledDensity = math.parse(params.mu).compile();
            // densityFunc = (x, y, z) => compiledDensity.evaluate({ x, y, z });

            const flatArray = new Float32Array(
                uniformColorData($densityColormap),
            );

            const shaderMaterial = new THREE.ShaderMaterial({
                vertexShader: plainVertexShader,
                fragmentShader: heatmapFragmentShader(
                    params.mu,
                    $densityColormap,
                    $vMin,
                    $vMax,
                ),
                uniforms: {
                    colorData: { value: flatArray },
                },
                side: THREE.DoubleSide,
            });

            box.material?.dispose();
            box.material = shaderMaterial;

            render();
        } else {
            if (box) {
                box.material = material;
                render();
            }
        }
    });

    const material = new THREE.MeshPhongMaterial({
        color: '#993588',
        shininess: 80,
        side: THREE.DoubleSide,
        vertexColors: false,
        transparent: false,
        opacity: 0.7,
    });

    $effect(() => {
        material.color.set(color);
        render();
    });

    let boxItemElement;
    /**
     * Close on mesh so reactive statement doesn't react when individual parameters change.
     */
    const flash = () => {
        flashDance(box, render);
        boxItemElement?.scrollIntoView({ behavior: 'smooth', block: 'start' });
        // console.log("I am scroll into view, I can't think of nothin' else.");
    };

    $effect(() => {
        if (selected) untrack(flash);
    });

    const whiteLineMaterial = new THREE.LineBasicMaterial({
        color: 0xffffff,
        linewidth: 2,
        side: THREE.BackSide,
    });

    const box = new THREE.Mesh(new THREE.BufferGeometry(), material);
    const solidGroup = new THREE.Group();

    box.name = uuid;

    const borders = new THREE.LineSegments(
        new THREE.BufferGeometry(),
        whiteLineMaterial,
    );

    solidGroup.add(box);
    solidGroup.add(borders);
    scene.add(solidGroup);

    const renameCoords = (s, coords) => {
        let stone;
        if (coords === 'rect') {
            stone = {
                theta: 'x',
                r: 'y',
                phi: 'y',
                rho: 'z',
            };
        } else if (coords === 'cyl') {
            stone = {
                x: 'theta',
                y: 'r',
                phi: 'r',
                rho: 'z',
            };
        } else {
            stone = {
                x: 'theta',
                y: 'phi',
                r: 'phi',
                z: 'rho',
            };
        }
        return math
            .parse(s)
            .transform((node) => {
                if (node.isSymbolNode) {
                    node.name = stone[node.name] || node.name;
                }
                return node;
            })
            .toString();
    };

    $effect(() => {
        if (params.coords) {
            untrack(() => {
                params.c = renameCoords(params.c, params.coords);
                params.d = renameCoords(params.d, params.coords);
                params.e = renameCoords(params.e, params.coords);
                params.f = renameCoords(params.f, params.coords);
            });
        }
    });

    // "compiled" versions of bounds
    let A, B, C, D, E, F;
    const updateRegion = () => {
        A = math.evaluate(params.a);
        B = math.evaluate(params.b);
        let geom;

        const cComp = math.parse(params.c).compile();
        const dComp = math.parse(params.d).compile();
        const eComp = math.parse(params.e).compile();
        const fComp = math.parse(params.f).compile();
        switch (params.coords) {
            case 'rect':
                C = (x) => cComp.evaluate({ x });
                D = (x) => dComp.evaluate({ x });
                E = (x, y) => eComp.evaluate({ x, y });
                F = (x, y) => fComp.evaluate({ x, y });
                geom = new RectangularSolidGeometry(A, B, C, D, E, F, nX, nX);
                break;
            case 'cyl':
                C = (theta) => cComp.evaluate({ theta });
                D = (theta) => dComp.evaluate({ theta });
                E = (r, theta) => eComp.evaluate({ r, theta });
                F = (r, theta) => fComp.evaluate({ r, theta });
                geom = new CylindricalSolidGeometry(
                    A,
                    B,
                    C,
                    D,
                    E,
                    F,
                    nX * 2,
                    nX / 2,
                );
                break;
            case 'spher':
                C = (theta) => cComp.evaluate({ theta });
                D = (theta) => dComp.evaluate({ theta });
                E = (theta, phi) => eComp.evaluate({ theta, phi });
                F = (theta, phi) => fComp.evaluate({ theta, phi });
                geom = new SphericalSolidGeometry(A, B, C, D, E, F, nX * 2, nX);
                break;
            default:
                console.error('Something went wrong with the coord system.');
                break;
        }

        geom.computeBoundingBox();
        geom.computeBoundingSphere();

        box.geometry?.dispose();
        box.geometry = geom;
        borders.geometry = new THREE.EdgesGeometry(geom, 40);

        onRenderObject(box);
        render();
    };

    // borders.visible = false;
    // console.log('boxes', box, borders);

    const chickenParms = (val, { a, b, c, d, coords }) => {
        let valuation;
        try {
            const [A, B, C, D, V] = math.parse([a, b, c, d, val]);

            const u = (A.evaluate() + B.evaluate()) / 2;
            const u1 = {};
            if (coords === 'rect') {
                u1['x'] = u;
            } else {
                u1['theta'] = u;
            }

            const v = (C.evaluate(u1) + D.evaluate(u1)) / 2;
            if (coords === 'rect') {
                u1['y'] = v;
            } else if (coords === 'cyl') {
                u1['r'] = v;
            } else {
                u1['phi'] = v;
            }
            valuation = V.evaluate(u1);
        } catch (error) {
            console.error('ParseError in evaluation.', error);
            return false;
        }
        if (Number.isFinite(valuation)) {
            return true;
        } else {
            console.error('Evaluation error. Incomplete expression, maybe.');
            return false;
        }
    };

    // Only run the update if the params have changed.
    // $: hashTag = checksum(JSON.stringify(params));
    $effect(() => {
        updateRegion();
        render();
    });

    render();
</script>

<!-- svelte-ignore a11y_no_static_element_interactions -->
<div class="boxItem" class:selected bind:this={boxItemElement}>
    <ObjHeader
        bind:minimize
        {toggleHide}
        {onClose}
        {color}
        onSelect={(e) => selectObject(uuid, !e.shiftKey)}
        {objHidden}
    >
        <Nametag bind:title />
    </ObjHeader>
    <div hidden={minimize}>
        <div class="threedemos-container container">
            <span class="box-1">Coordinates</span>
            <select class="box-2" bind:value={params.coords}>
                <option value="rect">Rectangular</option>
                <option value="cyl">Cylindrical</option>
                <option value="spher">Spherical</option>
            </select>

            {#each ['a', 'b'] as name}
                {#if name === 'b'}
                    <span class="box box-3">
                        <M
                            size="sm"
                            s={params.coords === 'rect'
                                ? `\\leq x \\leq`
                                : `\\leq \\theta \\leq`}
                        />
                    </span>
                {/if}
                <InputChecker
                    className="form-control form-control-sm {name === 'a'
                        ? 'box-1'
                        : 'box-4'}"
                    checker={(val) =>
                        Number.isFinite(math.parse(val).evaluate())}
                    value={params[name]}
                    {name}
                    cleared={(val) => {
                        params[name] = val;
                    }}
                />
            {/each}

            {#each ['c', 'd'] as name}
                {#if name === 'd'}
                    <span class="box box-3"
                        ><M
                            size="sm"
                            s={params.coords === 'rect'
                                ? `\\leq y \\leq`
                                : params.coords === 'cyl'
                                  ? `\\leq r \\leq`
                                  : `\\leq \\phi \\leq`}
                        />
                    </span>
                {/if}
                <InputChecker
                    className="form-control form-control-sm {name === 'c'
                        ? 'box-1'
                        : 'box-4'}"
                    checker={(val) => {
                        const A = math.evaluate(params.a);
                        const B = math.evaluate(params.b);
                        let parsedVal;
                        try {
                            parsedVal = math
                                .parse(val)
                                .evaluate(
                                    params.coords === 'rect'
                                        ? { x: (A + B) / 2 }
                                        : { theta: (A + B) / 2 },
                                );
                        } catch (e) {
                            console.error(e);
                            return false;
                        }
                        return Number.isFinite(parsedVal);
                    }}
                    value={params[name]}
                    {name}
                    cleared={(val) => {
                        params[name] = val;
                    }}
                />
            {/each}

            {#each ['e', 'f'] as name}
                {#if name === 'f'}
                    <span class="box box-3"
                        ><M
                            size="sm"
                            s={params.coords === 'spher'
                                ? `\\leq \\rho \\leq`
                                : `\\leq z \\leq`}
                        />
                    </span>
                {/if}
                <InputChecker
                    className="form-control form-control-sm {name === 'e'
                        ? 'box-1'
                        : 'box-4'}"
                    checker={chickenParms}
                    value={params[name]}
                    {name}
                    {params}
                    cleared={(val) => {
                        params[name] = val;
                    }}
                />
            {/each}

            <span class="box-1">Resolution</span>
            <input
                type="range"
                bind:value={nX}
                min="10"
                max="80"
                step="5"
                class="box box-2"
            />
            <span class="box-1"> Density </span>
            <label class="switch box box-3">
                <input
                    type="checkbox"
                    name="chooseDensity"
                    id="chooseDensity"
                    bind:checked={chooseDensity}
                />
                <span class="slider round"></span>
            </label>

            {#if chooseDensity}
                <span class="box-1"><M size="sm" s="\\mu(x,y,z) =" /></span>
                <InputChecker
                    value={params.mu}
                    checker={(val) => {
                        const pos = box.geometry.attributes.position.array;
                        const N = Math.round((Math.random() * pos.length) / 3);
                        const [x, y, z] = pos.slice(3 * N, 3 * N + 3);

                        let parsedVal;
                        try {
                            parsedVal = math.parse(val).evaluate({ x, y, z });
                        } catch (e) {
                            console.error(e);
                            return false;
                        }
                        return Number.isFinite(parsedVal);
                    }}
                    name={'mu'}
                    {params}
                    cleared={(val) => {
                        params.mu = val;
                        render();
                    }}
                />
                <div class="box colorbar-container">
                    <ColorBar
                        vMin={$vMin}
                        vMax={$vMax}
                        cmap={$densityColormap}
                    />
                </div>
            {:else}
                <span class="box box-2">
                    <input
                        type="color"
                        name="colorPicker"
                        id="colorPicker"
                        bind:value={color}
                        style="width:85%; padding: 1px 1px;"
                    />
                </span>
            {/if}
        </div>
    </div>
</div>

<style>
    .box-3 {
        color: white;
        vertical-align: middle;
        text-align: center;

        grid-column: 2 / 3;
    }
    input {
        color: black;
    }

    .colorbar-container {
        grid-column: 1 / -1;
        height: 2.5rem;
        margin-bottom: 5px;
    }
</style>
