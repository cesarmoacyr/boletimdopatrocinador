# Boletim do Patrocinador — Instruções

Guia para publicar uma nova edição mensal e manter a página
`https://ongrace.com/boletimdopatrocinador/`.

## Visão geral

- A página é **um único arquivo HTML** (`boletim-do-patrocinado.html`), colado como conteúdo numa página do WordPress.
- Cada edição é um **PDF** + uma **capa JPG**, ambos hospedados na Biblioteca de Mídia do WordPress.
- O botão **Visualizar** abre o PDF num modal, desenhado em `<canvas>` com **PDF.js** (sem a barra nativa do navegador com baixar/imprimir/Drive). Não usa mais DearFlip nem jQuery.
- O botão **Baixar** continua disponível, separado.
- O repositório GitHub (`cesarmoacyr/boletimdopatrocinador`, **público**) guarda uma cópia dos arquivos.

## Arquivos por edição

| Arquivo | Exemplo |
|---|---|
| PDF | `boletim-do-patrocinador-outubro-2026.pdf` |
| Capa | `capa-boletim-2026-10.jpg` |

O PDF original chega como `BOLETIM PATROCINADOR OUT_26 Flip.pdf`; renomeie (cópia) para o padrão acima.

## Passo a passo de uma nova edição

### 1. Preparar o PDF

```bash
cp "BOLETIM PATROCINADOR XXX_26 Flip.pdf" boletim-do-patrocinador-MES-ANO.pdf
```

Use o comando `python` (não `python3`, que no Windows aponta para outra instalação sem o pymupdf). Se faltar: `pip install pymupdf`.

### 2. Gerar a capa (página 1)

```python
import pymupdf
doc = pymupdf.open('boletim-do-patrocinador-MES-ANO.pdf')
pix = doc[0].get_pixmap(matrix=pymupdf.Matrix(1, 1))
pix.save('capa-boletim-ANO-MM.jpg', jpg_quality=80)
```

### 3. Inserir links reais no PDF

O PDF vem **sem links** (os "CLIQUE AQUI" e as redes sociais são só texto). O visualizador desenha por cima do canvas os links que existirem no PDF, então eles precisam ser inseridos no arquivo.

**Antes de rodar, olhe cada página.** O layout pode mudar (em outubro a pág. 2 tinha dois "CLIQUE AQUI", um por box; em setembro, só um). Cada botão deve apontar para o destino do **seu** box.

Destinos usados:

| Texto no PDF | Destino |
|---|---|
| "CLIQUE AQUI" do box Show da Fé / `www.ongrace.com` (pág. 2) | `https://www.ongrace.com` |
| "CLIQUE AQUI" do box `revistashowdafe` (pág. 2) | `https://www.revistashowdafe.com.br/mensagens` |
| "CLIQUE AQUI E FALE CONOSCO" (pág. 4) | `https://wa.me/552121413510` |
| Missionário — YouTube / Instagram / Facebook | `youtube.com/@missionariorrsoares` · `instagram.com/missionariorrsoares/` · `facebook.com/missionariorrsoares/` |
| IIGD — YouTube / Instagram / Facebook | `youtube.com/@IIGDoficial` · `instagram.com/iigdoficial/` · `facebook.com/iigdoficial/` |

Script (ajuste os índices de página se o layout mudar):

```python
import pymupdf, os
path = 'boletim-do-patrocinador-MES-ANO.pdf'
doc = pymupdf.open(path)
def link(p, r, url): p.insert_link({'kind': pymupdf.LINK_URI, 'from': r, 'uri': url})

# Pág. 2: cada botão no seu box (box da esquerda = x0 < 300)
p2 = doc[1]
for r in p2.search_for('CLIQUE AQUI'):
    link(p2, r, 'https://www.ongrace.com' if r.x0 < 300 else 'https://www.revistashowdafe.com.br/mensagens')
for r in p2.search_for('www.ongrace.com'): link(p2, r, 'https://www.ongrace.com')
for r in p2.search_for('www.revistashowdafe.com.br/mensagens'): link(p2, r, 'https://www.revistashowdafe.com.br/mensagens')

# Pág. 4: WhatsApp + redes sociais
p4 = doc[3]
for r in p4.search_for('CLIQUE AQUI E FALE CONOSCO'): link(p4, r, 'https://wa.me/552121413510')
words = p4.get_text('words')

def cluster_rows(ws, gap=5):
    ws = sorted(ws, key=lambda w: w[1]); rows, cur, last = [], [], None
    for w in ws:
        if last is not None and w[1] - last > gap: rows.append(cur); cur = []
        cur.append(w); last = w[1]
    if cur: rows.append(cur)
    return rows

def uniao(ws):
    r = pymupdf.Rect(ws[0][:4])
    for w in ws[1:]: r |= pymupdf.Rect(w[:4])
    return r

esq = cluster_rows([w for w in words if 90 < w[0] < 260 and 630 < w[1] < 700])
dir_ = cluster_rows([w for w in words if 300 < w[0] < 400 and 630 < w[1] < 700])
assert len(esq) == 3 and len(dir_) == 3
for linha, url in zip(esq, ['https://www.youtube.com/@missionariorrsoares', 'https://www.instagram.com/missionariorrsoares/', 'https://www.facebook.com/missionariorrsoares/']):
    link(p4, uniao(linha), url)
for linha, url in zip(dir_, ['https://www.youtube.com/@IIGDoficial', 'https://www.instagram.com/iigdoficial/', 'https://www.facebook.com/iigdoficial/']):
    link(p4, uniao(linha), url)

tmp = path + '.tmp'; doc.save(tmp); doc.close(); os.replace(tmp, path)
```

Por que agrupar por linhas (`cluster_rows`) e não por faixa de `y`: as 3 linhas de redes sociais ficam coladas (gap ≈ 0), e filtros do tipo `645 < y < 660` pegam a linha vizinha e geram links sobrepostos. Além disso, "ﬁ" em "IIGDoﬁcial" é uma ligadura que o PyMuPDF separa em duas palavras ("IIGDoﬁ" + "cial"); por isso se une o retângulo das palavras da linha, em vez de buscar o texto.

Conferir o resultado:

```python
d = pymupdf.open(path)
for i, p in enumerate(d): print(i + 1, [(l['uri'], [round(x) for x in l['from']]) for l in p.get_links()])
```

### 4. Subir no WordPress

1. Biblioteca de Mídia → enviar o PDF e a capa JPG.
2. Copiar as URLs geradas (normalmente `https://ongrace.com/wp-content/uploads/AAAA/MM/arquivo`).

**Importante:** o PDF e a página precisam estar no **mesmo domínio** (`ongrace.com`). O PDF.js busca o arquivo via `fetch`, e outro domínio seria bloqueado por CORS.

### 5. Atualizar o HTML

No final do arquivo, no array `edicoes`, adicione a nova edição **no topo** (a primeira vira o destaque do hero):

```js
const edicoes = [
  { mes: 9, ano: 2026, arquivo: 'https://ongrace.com/wp-content/uploads/2026/10/boletim-do-patrocinador-outubro-2026.pdf', capa: 'https://ongrace.com/wp-content/uploads/2026/10/capa-boletim-2026-10.jpg' }, // Outubro/2026
  { mes: 8, ano: 2026, ... }, // Setembro/2026
];
```

- `mes` começa em 0: janeiro = 0 … dezembro = 11.
- `arquivo: null` mostra o card como "Em breve" com botões desativados.
- Os textos gerais (hero, "sobre", seções) são os mesmos todo mês e não precisam mudar.

### 6. Testar localmente

Abrir o HTML por `file://` **não funciona** para testar o PDF (o navegador bloqueia o acesso a arquivos locais). Suba um servidor na pasta:

```bash
cd "caminho\boletim do patrocinador"
python -m http.server 8790
```

Abra `http://localhost:8790/boletim-do-patrocinado.html` e confira:

- hero mostra o mês novo; a grade lista as edições;
- **Visualizar** abre o modal, as 4 páginas passam sem erro e os links da pág. 2 e 4 respondem;
- console sem erros (as capas e PDFs do WordPress só carregam depois do upload).

Para testar um PDF local no modal, rode no console: `abrirVisualizador('boletim-do-patrocinador-MES-ANO.pdf')`.

### 7. Publicar

Colar o HTML atualizado no conteúdo da página do WordPress (bloco HTML personalizado).

### 8. GitHub

Commitar **só os arquivos da edição** (HTML, PDF e capa novos), sem `git add -A`:

```bash
git add boletim-do-patrocinado.html boletim-do-patrocinador-MES-ANO.pdf capa-boletim-ANO-MM.jpg
git commit -m "Adiciona edição de MES/ANO"
git push origin main
```

Há alterações antigas pendentes no repositório local (PDF e capa de agosto, fotos de testemunhos apagadas, `thumb-...jpeg`, o PDF "Flip" original). Elas não fazem parte do fluxo mensal e ficam de fora dos commits.

## Detalhes técnicos que evitam bugs

- **Nunca usar `&&` dentro do `<script>`.** O WordPress converte `&&` em `&#038;&#038;` e quebra o script inteiro, sem erro visível. Use `if` aninhado.
- **Zoom nítido no mobile:** o canvas é renderizado em resolução alta (densidade de pixel × 2,5, teto de 2800 px) e exibido menor via CSS. Sem isso o texto fica borrado ao dar pinça.
- **Camada de links:** como o PDF é desenhado em canvas, os links precisam ser recriados como `<a>` sobre o canvas (`renderizarLinksPdf`). Só links de URI são tratados.
- **PDF.js** é carregado do cdnjs, versão fixa `3.11.174`, no `<head>`. Se trocar a versão, troque também a URL do `workerSrc` no script.
- **Menu e hero:** o card da edição é movido no DOM (e não por `grid-template-areas`) entre mobile e desktop, para evitar um espaço vazio causado pelo CSS Grid.
- **SEO:** título, descrição e Open Graph que valem são os do **Yoast** na edição da página do WordPress; as tags comentadas no `<head>` do HTML só servem se o arquivo for hospedado sozinho. O `<h1>` e o `<h2>` estruturais usam a classe `.sr-only`.
- **Proteção contra baixar/imprimir:** o modal elimina os atalhos de um clique (baixar, imprimir, Drive) e bloqueia o clique-direito no canvas, mas print de tela e ferramentas de desenvolvedor continuam possíveis. Não existe bloqueio total em conteúdo exibido no navegador.
- **Windows:** imprimir texto com ligaduras ("ﬁ") no console do Python gera `UnicodeEncodeError`. Grave a saída em arquivo UTF-8 e leia o arquivo.

## Segurança — pontos em aberto

Levantados na auditoria e ainda sem decisão:

1. Interpolação `${}` sem escape em `criarBotoes()`/`renderEdicoes()` (baixo risco hoje, pois os dados são fixos no próprio HTML).
2. O link do botão **Cadastro** carrega parâmetros de rastreamento (`_gl=...`); vale trocar pela URL limpa.
3. O repositório é **público** e contém fotos de pessoas dos testemunhos (já apagadas do disco, mas ainda no histórico do Git). Decidir entre só apagar ou reescrever o histórico.
