# App de Filmes e Séries (PWA) para Android

Você vai criar 5 arquivos na mesma pasta: index.html, style.css, app.js, manifest.json e sw.js. Depois, é só hospedar de graça no GitHub Pages e instalar pelo Chrome.

**Importante:** no arquivo app.js, troque SUA\_CHAVE\_AQUI pela sua chave do TMDB.

**Aviso:** o app mostra catálogo, sinopses, elenco e trailers oficiais do YouTube. Ele não reproduz filmes ou séries completos.

## Arquivo 1: index.html

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <meta name="theme-color" content="#0f0f14">
  <link rel="manifest" href="manifest.json">
  <link rel="stylesheet" href="style.css">
  <title>Meu Cine</title>
</head>
<body>
  <header>
    <h1 id="logo">Meu Cine</h1>
    <input id="busca" type="search" placeholder="Buscar filmes e séries...">
  </header>

  <nav>
    <button class="aba ativa" data-aba="inicio">Início</button>
    <button class="aba" data-aba="filmes">Filmes</button>
    <button class="aba" data-aba="series">Séries</button>
    <button class="aba" data-aba="favoritos">Favoritos</button>
  </nav>

  <main id="conteudo"></main>

  <div id="detalhes" class="oculto"></div>

  <script src="app.js"></script>
</body>
</html>
```

## Arquivo 2: style.css

```css
* { box-sizing: border-box; margin: 0; padding: 0; }
body { background: #0f0f14; color: #fff; font-family: system-ui, sans-serif; padding-bottom: 40px; }
header { padding: 14px; position: sticky; top: 0; background: #0f0f14; z-index: 5; }
#logo { font-size: 22px; color: #e50914; margin-bottom: 10px; cursor: pointer; }
#busca { width: 100%; padding: 12px; border-radius: 10px; border: none; background: #22222c; color: #fff; font-size: 16px; }
nav { display: flex; gap: 8px; padding: 0 14px 10px; overflow-x: auto; }
.aba { background: #22222c; color: #ccc; border: none; padding: 8px 16px; border-radius: 20px; font-size: 14px; white-space: nowrap; }
.aba.ativa { background: #e50914; color: #fff; }
h2 { font-size: 18px; padding: 14px 14px 8px; }
.carrossel { display: flex; gap: 10px; overflow-x: auto; padding: 0 14px; scroll-snap-type: x proximity; }
.carrossel .card { flex: 0 0 120px; scroll-snap-align: start; }
.grade { display: grid; grid-template-columns: repeat(auto-fill, minmax(110px, 1fr)); gap: 10px; padding: 0 14px; }
.card img { width: 100%; border-radius: 8px; aspect-ratio: 2/3; object-fit: cover; background: #22222c; }
.card p { font-size: 12px; margin-top: 4px; color: #ddd; }
#detalhes { position: fixed; inset: 0; background: #0f0f14; overflow-y: auto; z-index: 10; }
.oculto { display: none; }
.topo-detalhe { position: relative; }
.topo-detalhe img.fundo { width: 100%; max-height: 260px; object-fit: cover; }
.voltar { position: absolute; top: 12px; left: 12px; background: rgba(0,0,0,.6); color: #fff; border: none; padding: 10px 14px; border-radius: 20px; font-size: 14px; }
.info { padding: 14px; }
.info h1 { font-size: 22px; margin-bottom: 6px; }
.meta { color: #aaa; font-size: 13px; margin-bottom: 10px; }
.info p { line-height: 1.5; font-size: 14px; color: #ddd; }
.botao { background: #e50914; color: #fff; border: none; padding: 12px 18px; border-radius: 10px; font-size: 15px; margin: 12px 8px 12px 0; }
.botao.sec { background: #22222c; }
.trailer { width: calc(100% - 28px); margin: 0 14px; aspect-ratio: 16/9; border: none; border-radius: 10px; }
.vazio { padding: 30px 14px; color: #888; text-align: center; }
```

## Arquivo 3: app.js

```javascript
const CHAVE = 1500e6e7dfd3f6b239af8e8c0c610dd3;
const API = 'https://api.themoviedb.org/3';
const IMG = 'https://image.tmdb.org/t/p/w342';
const FUNDO = 'https://image.tmdb.org/t/p/w780';
const IDIOMA = 'pt-BR';

const conteudo = document.getElementById('conteudo');
const detalhes = document.getElementById('detalhes');
const busca = document.getElementById('busca');
let abaAtual = 'inicio';

function favoritos() {
  return JSON.parse(localStorage.getItem('favoritos') || '[]');
}
function salvarFavoritos(lista) {
  localStorage.setItem('favoritos', JSON.stringify(lista));
}

async function chamar(caminho, extra = '') {
  const url = `${API}${caminho}?api_key=${CHAVE}&language=${IDIOMA}${extra}`;
  const resp = await fetch(url);
  if (!resp.ok) throw new Error('Erro na API');
  return resp.json();
}

function cardHTML(item, tipo) {
  const titulo = item.title || item.name || '';
  const foto = item.poster_path ? IMG + item.poster_path : '';
  return `<div class="card" onclick="abrirDetalhes(${item.id}, '${tipo}')">
    <img loading="lazy" src="${foto}" alt="${titulo}">
    <p>${titulo}</p>
  </div>`;
}

function secao(titulo, itens, tipo, carrossel = true) {
  const classe = carrossel ? 'carrossel' : 'grade';
  return `<h2>${titulo}</h2><div class="${classe}">${itens.map(i => cardHTML(i, i.media_type || tipo)).join('')}</div>`;
}

async function mostrarInicio() {
  conteudo.innerHTML = '<p class="vazio">Carregando...</p>';
  try {
    const [alta, filmes, series, nota] = await Promise.all([
      chamar('/trending/all/week'),
      chamar('/movie/popular'),
      chamar('/tv/popular'),
      chamar('/movie/top_rated')
    ]);
    conteudo.innerHTML =
      secao('Em alta', alta.results, 'movie') +
      secao('Filmes populares', filmes.results, 'movie') +
      secao('Séries populares', series.results, 'tv') +
      secao('Mais bem avaliados', nota.results, 'movie');
  } catch (e) {
    conteudo.innerHTML = '<p class="vazio">Erro ao carregar. Confira sua chave e a internet.</p>';
  }
}

async function mostrarLista(tipo) {
  conteudo.innerHTML = '<p class="vazio">Carregando...</p>';
  try {
    const [pop, nota, novos] = await Promise.all([
      chamar(`/${tipo}/popular`),
      chamar(`/${tipo}/top_rated`),
      chamar(tipo === 'movie' ? '/movie/now_playing' : '/tv/on_the_air')
    ]);
    conteudo.innerHTML =
      secao('Populares', pop.results, tipo) +
      secao('Mais bem avaliados', nota.results, tipo) +
      secao('Lançamentos', novos.results, tipo);
  } catch (e) {
    conteudo.innerHTML = '<p class="vazio">Erro ao carregar.</p>';
  }
}

function mostrarFavoritos() {
  const lista = favoritos();
  if (!lista.length) {
    conteudo.innerHTML = '<p class="vazio">Você ainda não tem favoritos.</p>';
    return;
  }
  conteudo.innerHTML = secao('Meus favoritos', lista, 'movie', false);
}

async function abrirDetalhes(id, tipo) {
  detalhes.classList.remove('oculto');
  detalhes.innerHTML = '<p class="vazio">Carregando...</p>';
  try {
    const d = await chamar(`/${tipo}/${id}`, '&append_to_response=videos,credits,similar');
    const titulo = d.title || d.name;
    const ano = (d.release_date || d.first_air_date || '').slice(0, 4);
    const duracao = d.runtime ? d.runtime + ' min' : (d.number_of_seasons ? d.number_of_seasons + ' temporada(s)' : '');
    const generos = (d.genres || []).map(g => g.name).join(', ');
    const elenco = (d.credits?.cast || []).slice(0, 8).map(a => a.name).join(', ');
    const trailer = (d.videos?.results || []).find(v => v.site === 'YouTube' && v.type === 'Trailer');
    const ehFav = favoritos().some(f => f.id === d.id);
    const semelhantes = (d.similar?.results || []).slice(0, 12);
    const item = { id: d.id, title: titulo, poster_path: d.poster_path, media_type: tipo };

    detalhes.innerHTML = `
      <div class="topo-detalhe">
        <button class="voltar" onclick="fecharDetalhes()">← Voltar</button>
        ${d.backdrop_path ? `<img class="fundo" src="${FUNDO + d.backdrop_path}">` : ''}
      </div>
      <div class="info">
        <h1>${titulo}</h1>
        <div class="meta">${ano} • ${duracao} • ⭐ ${d.vote_average.toFixed(1)}<br>${generos}</div>
        <button class="botao" id="btnFav">${ehFav ? '★ Nos favoritos' : '☆ Favoritar'}</button>
        <p>${d.overview || 'Sem sinopse disponível.'}</p>
        <p class="meta" style="margin-top:10px"><b>Elenco:</b> ${elenco}</p>
      </div>
      ${trailer ? `<iframe class="trailer" src="https://www.youtube.com/embed/${trailer.key}" allowfullscreen></iframe>` : ''}
      ${semelhantes.length ? secao('Semelhantes', semelhantes, tipo) : ''}
    `;
    document.getElementById('btnFav').onclick = () => alternarFavorito(item);
    detalhes.scrollTop = 0;
  } catch (e) {
    detalhes.innerHTML = '<button class="voltar" onclick="fecharDetalhes()">← Voltar</button><p class="vazio">Erro ao carregar detalhes.</p>';
  }
}

function alternarFavorito(item) {
  let lista = favoritos();
  const existe = lista.some(f => f.id === item.id);
  lista = existe ? lista.filter(f => f.id !== item.id) : [...lista, item];
  salvarFavoritos(lista);
  document.getElementById('btnFav').textContent = existe ? '☆ Favoritar' : '★ Nos favoritos';
  if (abaAtual === 'favoritos') mostrarFavoritos();
}

function fecharDetalhes() {
  detalhes.classList.add('oculto');
  detalhes.innerHTML = '';
}

let atraso;
busca.addEventListener('input', () => {
  clearTimeout(atraso);
  const termo = busca.value.trim();
  if (!termo) { trocarAba(abaAtual); return; }
  atraso = setTimeout(async () => {
    try {
      const r = await chamar('/search/multi', `&query=${encodeURIComponent(termo)}`);
      const itens = r.results.filter(i => i.media_type !== 'person');
      conteudo.innerHTML = itens.length
        ? secao('Resultados', itens, 'movie', false)
        : '<p class="vazio">Nada encontrado.</p>';
    } catch (e) {
      conteudo.innerHTML = '<p class="vazio">Erro na busca.</p>';
    }
  }, 400);
});

function trocarAba(aba) {
  abaAtual = aba;
  busca.value = '';
  document.querySelectorAll('.aba').forEach(b => b.classList.toggle('ativa', b.dataset.aba === aba));
  if (aba === 'inicio') mostrarInicio();
  if (aba === 'filmes') mostrarLista('movie');
  if (aba === 'series') mostrarLista('tv');
  if (aba === 'favoritos') mostrarFavoritos();
}

document.querySelectorAll('.aba').forEach(b => b.onclick = () => trocarAba(b.dataset.aba));
document.getElementById('logo').onclick = () => trocarAba('inicio');

if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('sw.js');
}

trocarAba('inicio');
```

## Arquivo 4: manifest.json

```json
{
  "name": "Meu Cine",
  "short_name": "Meu Cine",
  "start_url": "./index.html",
  "display": "standalone",
  "background_color": "#0f0f14",
  "theme_color": "#0f0f14",
  "icons": [
    { "src": "icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

Você também precisa de duas imagens quadradas chamadas icon-192.png e icon-512.png na mesma pasta. Pode ser qualquer logo simples, e eu te ajudo a fazer uma se quiser.

## Arquivo 5: sw.js

```javascript
const CACHE = 'meu-cine-v1';
const ARQUIVOS = ['./', './index.html', './style.css', './app.js', './manifest.json'];

self.addEventListener('install', e => {
  e.waitUntil(caches.open(CACHE).then(c => c.addAll(ARQUIVOS)));
});

self.addEventListener('fetch', e => {
  if (e.request.url.includes('api.themoviedb.org') || e.request.url.includes('image.tmdb.org')) return;
  e.respondWith(caches.match(e.request).then(r => r || fetch(e.request)));
});
```

## Como hospedar e instalar

1. Crie uma conta gratuita em github.com.
2. Crie um repositório novo, público, com o nome que quiser.
3. Envie os 5 arquivos e as 2 imagens de ícone para o repositório, usando a opção Add file, Upload files.
4. Vá em Settings, depois Pages, escolha a branch main e a pasta root, e salve.
5. Em alguns minutos, seu site fica disponível em seunome.github.io/nome-do-repositorio.
6. Abra esse endereço no Chrome do Android, toque nos três pontinhos e escolha Instalar aplicativo ou Adicionar à tela inicial.

Pronto: o ícone aparece na tela inicial e o app abre em tela cheia, como um aplicativo de verdade.
