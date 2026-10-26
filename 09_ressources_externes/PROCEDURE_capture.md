---
title: "Procédure — capture du canal Discord"
type: meta
tags:
  - beyond-nudge
  - LDD
  - discord
  - methode
created: 2026-09-07
---

# Procédure — capture du canal Discord

Comment refaire ou mettre à jour la capture décrite dans DISCORD. Écrit pour être suivi depuis zéro, dans n'importe quelle conversation.

## Contexte technique

Il n'existe **pas de connecteur Discord** (vérifié dans le répertoire de connecteurs). L'accès passe par **Claude in Chrome**, le navigateur réel d'Evann, où la session Discord est déjà ouverte.

> Le navigateur intégré de Claude a un profil séparé et n'est **pas** connecté à Discord. Ne pas perdre de temps avec lui. Ne jamais saisir de mot de passe.

**Canal** : `#beyond-nudge`, serveur *Laboratoire des déviations écologiques*
`https://discord.com/channels/1101188745153216625/1527254548744503356`

**Borne** : le champ `derniere_capture:` dans le frontmatter de  la note DISCORD. Ne traiter que les messages postés après cette date.

## Étapes

### 1. Ouvrir le canal

`tabs_context_mcp {createIfEmpty: true}` puis `navigate` vers l'URL. Vérifier par screenshot que la session est connectée. Si Discord affiche l'écran de connexion, s'arrêter et le signaler.

### 2. Charger l'historique

Discord ne garde en mémoire que les messages visibles. Remonter le fil jusqu'à dépasser la borne :

```js
const sc=[...document.querySelectorAll('div')].find(d=>d.className&&/scroller/.test(d.className)
  &&d.scrollHeight>d.clientHeight+50&&d.querySelector('li[id^="chat-messages-"]'));
let prev=-1;
for(let i=0;i<40;i++){ sc.scrollTop=0; await new Promise(r=>setTimeout(r,800));
  const n=document.querySelectorAll('li[id^="chat-messages-"]').length; if(n===prev) break; prev=n; }
document.querySelectorAll('li[id^="chat-messages-"]').length
```

### 3. Extraire

```js
window.__strip=s=>s.replace(/(https?:\/\/[^\s]+?)\?[^\s]*/g,'$1')
                   .replace(/[?&][A-Za-z0-9_%.-]+=[^\s]*/g,'');
const clean=u=>{try{const x=new URL(u);return x.origin+x.pathname;}catch(e){return '';}};
window.__all=[...document.querySelectorAll('li[id^="chat-messages-"]')].map(li=>{
 const t=li.querySelector('time[datetime]'); if(!t) return null;
 const a=li.querySelector('[class*="username"]'); const c=li.querySelector('[id^="message-content-"]');
 const links=[...new Set([...li.querySelectorAll('a[href]')].map(x=>clean(x.href))
   .filter(h=>h&&!/discord(app)?\.(com|net)/.test(h)&&!/\.(jpg|jpeg|png|gif|svg|webp)$/i.test(h)))];
 return {dt:t.getAttribute('datetime'), au:a?a.innerText.trim():'',
         tx:window.__strip(c?c.innerText:''), lk:links};
}).filter(Boolean);
const byDay={}; window.__all.forEach(m=>{const d=m.dt.slice(0,10); byDay[d]=(byDay[d]||0)+1;}); byDay
```

## Les deux pièges

Ils ont coûté du temps la première fois. Ils sont structurels, ils se reproduiront.

**1. Le filtre de sécurité bloque les query strings.** Toute valeur retournée contenant `?param=...`, un cookie ou une clé déclenche `[BLOCKED: Cookie/query string data]` et l'appel ne renvoie rien. D'où `__strip` sur les textes et `clean` sur les liens. Ne pas retourner non plus la liste brute des auteurs (bloquée comme clé sensible) — compter les auteurs plutôt que les lister.

**2. Le résultat JavaScript est tronqué à ~1000 caractères par appel.** Pour un volume important, construire une chaîne unique dans la page et la tirer par tranches :

```js
window.__S = window.__all.filter(m=>m.dt.slice(0,10) >= '<BORNE>')
  .map((m,i)=>'@@'+i+'\n'+m.tx+'\nL: '+m.lk.join(' | ')).join('\n\n');
window.__P = o => window.__S.slice(o,o+950);
window.__S.length
```

Puis `window.__P(0)`, `window.__P(950)`, `window.__P(1900)`… En cours de route, repérer les doublons (reposts à l'identique, fréquents) et sauter directement au-delà avec `window.__S.indexOf('\n@@N\n')`.

> Un export en fichier téléchargé serait plus rapide, mais **Evann a explicitement préféré les tranches successives** (pas de permission supplémentaire à accorder). Ne pas reproposer l'export sans qu'il le demande.

## Écrire les notes

**Une note par date**, nommée `AAAA-MM-JJ.md`, écrite avec `device_bash` directement dans ce dossier.

Utiliser un **heredoc bash cité** (`cat > fichier.md <<'MDEOF' … MDEOF`), **pas** du formatage Python `%` : les URLs contiennent des `%20`, `%C3%A9` qui le cassent (erreur rencontrée la première fois).

Frontmatter :

```yaml
---
title: "<titre parlant, pas la date>"
type: discord-capture
source: "#beyond-nudge — Laboratoire des déviations écologiques"
channel: "<url du canal>"
author: <auteur>
date: AAAA-MM-JJ
messages: <n>
tags: [beyond-nudge, LDD, discord, + tags thématiques]
status: to-triage
captured: <date du jour>
---
```

`status: to-triage` n'est pas décoratif : le graphe Obsidian peut être filtré dessus (`-["status":"to-triage"]`), donc une note mal renseignée réapparaît à tort dans le graphe.

Corps : texte intégral des messages en blockquote, liens en liste, puis la section « Rattachement à la revue ».

## Le rattachement à la revue — le cœur du travail

La capture brute n'a aucun intérêt en soi. Pour chaque message, dire **à quelle case de structure_revue_litterature il alimente** (Règle 6 de Instructions) :

1. Histoire du mouvement / typologie des approches
2. Bilan dans les transitions (approche directe)
3. Bilan dans des domaines connexes (approche indirecte)
4. Contre-propositions et leurs limites
5. Critiques encore non adressées et perspectives

Et appliquer les exigences du projet :

- **Bilan en positif ET en négatif** — ce que l'intervention démontre effectivement, pas seulement ses limites.
- **Traiter les contre-propositions avec la même exigence critique que les nudges.** Une méthode qui dit *comment faire* sans évaluation comparative, c'est une limite à nommer explicitement.
- Signaler les recoupements avec le vault existant (`00_notes_papers/`, `03_mapping_ideas/`, `06_auteurs/`).
- Marquer ⭐ les entrées qui justifient une note de papier à part entière.

Quand une journée contient beaucoup de messages, regrouper **par case de la revue** plutôt que par ordre chronologique, en gardant une numérotation `@@N` stable pour la traçabilité.

## Toujours signaler les limites

À rappeler dans chaque note :

- Les textes sont souvent **repris de posts LinkedIn de leurs auteurs** — résumés promotionnels, pas des abstracts revus. À vérifier avant citation.
- Les liens raccourcis (`lnkd.in`) et les pages d'accueil de revue sont à résoudre vers leur source.
- Pièces jointes et images non récupérées.

## Finir

Mettre à jour la note DISCORD : nouvelles lignes du tableau **et** le champ `derniere_capture:`, qui est la borne de la prochaine exécution.
