<script>
    import Badge from './Badge.svelte';
    // import M from '../M.svelte';
    import { derived } from 'svelte/store';

    let N = $state(2);

    let s = $derived.by(() => {
        let out = '\\sin x = x + ';
        for (let i = 1; i < N; i++) {
            out += `\\frac{1}{${2 * i + 1}!} x^{${2 * i + 1}} + `;
        }

        out += `O(x^{${2 * N + 1}})`;
        return out;
    });
</script>

<p>Welcome to 3Demos, a tool for mathematics visualizations.</p>
<p>
    Click <Badge>Objects</Badge> &rarr; <Badge type="primary">Add Object</Badge>
    below to dive right in. Else, choose a topic above to learn more.
</p>

<!-- <input type="range" min="1" max="10" bind:value={N} />
<p> Test <code> M.svelte </code>
    Hello <M display {s} /> how are you, <M s={`e^{i\\pi x}`} />?
</p>
 -->
