<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, readAs, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // `filed`: where each problem is tracked now, problem → { label, href }.
  let { ticket, filed }: { ticket: number; filed: Record<string, { label: string; href: string }> } = $props();

  // Counted the desk's way (causes.ts): every report under its cause. The one-day moves the desk
  // read in desk-notes.json stay counted where they're filed, and get a row of their own too.
  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const groups = $derived(groupByCause(merged, tenantOf));
  const moved = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));
  const movedDays = $derived(moved.map((r) => r.receivedAt.slice(0, 10)).sort());
  const problems = $derived(groups.length + (moved.length ? 1 : 0));
  const tags = (rs: Report[]) =>
    Object.entries(rs.reduce<Record<string, number>>((n, r) => ({ ...n, [r.meta.intakeTag]: (n[r.meta.intakeTag] ?? 0) + 1 }), {}))
      .sort((a, b) => b[1] - a[1])
      .map(([tag, n]) => `${tag} ×${n}`)
      .join(' · ');
  const link = (problem: string) => filed[problem];
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} ·
  <strong>{problems}</strong>
  {problems === 1 ? 'problem' : 'problems'}
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>intake tagged them</th><th>tenants</th><th>seen</th><th>filed as</th></tr>
  </thead>
  <tbody>
    {#each groups as g (g.cause)}
      {@const f = link(g.cause)}
      <tr class:unfiled={!f}>
        <td><code>{g.cause}</code></td>
        <td>{g.reports.length}</td>
        <td class="tags">{tags(g.reports)}</td>
        <td>{g.tenants}</td>
        <td>{g.firstSeen} → {g.lastSeen}</td>
        <td>{#if f}<a href={f.href}>{f.label}</a>{:else}no ticket yet{/if}</td>
      </tr>
    {/each}
    {#if moved.length}
      {@const f = link('one-day')}
      <tr class:unfiled={!f}>
        <td><code>one-day</code></td>
        <td>{moved.length}</td>
        <td class="tags">{tags(moved)}</td>
        <td>{new Set(moved.map((r) => r.tenant)).size}</td>
        <td>{movedDays[0]} → {movedDays[movedDays.length - 1]}</td>
        <td>{#if f}<a href={f.href}>{f.label}</a>{:else}no ticket yet{/if}</td>
      </tr>
    {/if}
  </tbody>
</table>

{#each groups as g (g.cause)}
  {@const n = readAs(g, notes.oneDayMove)}
  {#if n}
    <p class="note">{n} of the <code>{g.cause}</code> reports are the one-day moves: counted under <code>{g.cause}</code>, where the desk files them by tenant, and again as <code>one-day</code>.</p>
  {/if}
{/each}

<style>
  .sum { font-size: 1.1rem; }
  .tags { color: var(--muted); font-size: 0.85em; }
  .note { color: var(--muted); font-size: 0.85em; }
  .unfiled td { background: var(--warn-bg); }
  .unfiled td:last-child { color: var(--warn); font-weight: 600; }
</style>
