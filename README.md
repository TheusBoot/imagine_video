# 🎬 Video Automator — Guia Completo

> Projeto para gerar vídeos automaticamente a partir de um tema, usando IA, APIs de mídia e FFmpeg.

---

## Sumário

1. [Visão Geral](#visão-geral)
2. [Arquitetura do Sistema](#arquitetura-do-sistema)
3. [Stack Tecnológica](#stack-tecnológica)
4. [Estrutura de Pastas](#estrutura-de-pastas)
5. [Etapa 1 — Parsing do Tema com IA](#etapa-1--parsing-do-tema-com-ia)
6. [Etapa 2 — Busca e Download de Vídeos](#etapa-2--busca-e-download-de-vídeos)
7. [Etapa 3 — Edição Automática dos Clipes](#etapa-3--edição-automática-dos-clipes)
8. [Etapa 4 — Busca e Mix de Música](#etapa-4--busca-e-mix-de-música)
9. [Etapa 5 — Renderização Final](#etapa-5--renderização-final)
10. [CLI — Ponto de Entrada](#cli--ponto-de-entrada)
11. [Variáveis de Ambiente](#variáveis-de-ambiente)
12. [Dependências](#dependências)
13. [Roadmap de Evolução](#roadmap-de-evolução)
14. [Dicas e Armadilhas Comuns](#dicas-e-armadilhas-comuns)

---

## Visão Geral

O **Video Automator** é um sistema em **Node.js** que recebe um tema como entrada (ex: `"animais correndo na floresta com música clássica"`) e produz automaticamente um vídeo final completo com:

- 4 a 5 clipes de vídeo relevantes ao tema
- Transições suaves entre os clipes
- Trilha sonora adequada ao mood
- Color grading automático
- Fade in e fade out no áudio e no vídeo

O fluxo completo é orquestrado por um pipeline de 5 etapas, onde a IA (Claude API) atua como o "cérebro" que planeja o vídeo antes de qualquer processamento começar.

---

## Arquitetura do Sistema

```
Tema (string)
     │
     ▼
┌─────────────────────┐
│  1. IA (Claude API) │  → Extrai keywords, mood, estilo musical, duração de cada clipe
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  2. Busca + Download│  → Pexels API, Pixabay API, yt-dlp
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  3. Edição de Clipes│  → FFmpeg: corte, resize, color grade, transições
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  4. Música          │  → Free Music Archive, Pixabay Music, mix com FFmpeg
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  5. Render Final    │  → Junta tudo, exporta MP4
└─────────────────────┘
           │
           ▼
    output/video.mp4
```

---

## Stack Tecnológica

| Camada | Tecnologia | Motivo |
|---|---|---|
| Runtime | Node.js 20+ | Melhor ecossistema para automação de mídia |
| IA / LLM | Claude API (Anthropic) | Extração de keywords, planejamento do vídeo |
| Busca de vídeos | Pexels API + Pixabay API | Gratuitas, licença livre, alta qualidade |
| Download YouTube | yt-dlp (via child_process) | Mais robusto que youtube-dl |
| Edição de vídeo | FFmpeg via fluent-ffmpeg | Padrão da indústria, extremamente poderoso |
| Edição declarativa | Editly (opcional) | Simplifica montagem de clipes com JSON |
| Música | Free Music Archive API + Pixabay Music | Gratuitas, sem copyright |
| HTTP client | axios | Simples e confiável |
| Variáveis de ambiente | dotenv | Gerenciamento de chaves de API |
| CLI | commander.js | Interface de linha de comando elegante |

---

## Estrutura de Pastas

```
video-automator/
│
├── src/
│   ├── ai/
│   │   └── planner.js          # Chama Claude API, retorna plano do vídeo em JSON
│   │
│   ├── search/
│   │   ├── pexels.js           # Busca vídeos na API do Pexels
│   │   ├── pixabay.js          # Busca vídeos na API do Pixabay
│   │   └── downloader.js       # Baixa os arquivos de vídeo para /tmp
│   │
│   ├── editor/
│   │   ├── trimmer.js          # Corta os clipes no tempo certo
│   │   ├── grader.js           # Aplica color grading com FFmpeg
│   │   └── transitions.js      # Gera transições entre clipes
│   │
│   ├── music/
│   │   ├── finder.js           # Busca música compatível com o mood
│   │   └── mixer.js            # Mix do áudio com o vídeo via FFmpeg
│   │
│   ├── renderer/
│   │   └── render.js           # Junta todos os clipes e exporta o vídeo final
│   │
│   └── utils/
│       ├── logger.js           # Logs coloridos no terminal
│       └── cleanup.js          # Remove arquivos temporários
│
├── tmp/                        # Vídeos e áudios baixados (gerado automaticamente)
├── output/                     # Vídeos finais exportados (gerado automaticamente)
│
├── .env                        # Chaves de API (não commitar!)
├── .env.example                # Exemplo de variáveis necessárias
├── .gitignore
├── package.json
└── index.js                    # CLI principal
```

---

## Etapa 1 — Parsing do Tema com IA

O módulo `planner.js` envia o tema para a Claude API e recebe um plano estruturado em JSON com tudo que o pipeline vai precisar.

### `src/ai/planner.js`

```javascript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

export async function planVideo(theme) {
  const prompt = `
    Você é um diretor de vídeo automático. Com base no tema abaixo, gere um plano de vídeo em JSON.
    
    Tema: "${theme}"
    
    Retorne APENAS um JSON válido com esta estrutura (sem markdown, sem explicações):
    {
      "title": "título descritivo do vídeo",
      "duration_seconds": 60,
      "clips": [
        {
          "order": 1,
          "search_query": "query em inglês para buscar o vídeo",
          "duration_seconds": 12,
          "color_grade": "warm | cool | neutral | vibrant",
          "description": "descrição do que deve aparecer neste clipe"
        }
      ],
      "music": {
        "mood": "classical | ambient | upbeat | calm | dramatic | epic",
        "search_query": "query em inglês para buscar a música",
        "volume": 0.4
      },
      "transitions": "fade | dissolve | cut",
      "aspect_ratio": "16:9",
      "resolution": "1920x1080"
    }
    
    Regras:
    - Gere entre 4 e 5 clipes
    - A soma de duration_seconds dos clipes deve ser igual ao duration_seconds total
    - As search_queries devem ser específicas e em inglês
    - Escolha color_grade de acordo com o mood do tema
  `;

  const response = await client.messages.create({
    model: "claude-opus-4-5",
    max_tokens: 1024,
    messages: [{ role: "user", content: prompt }],
  });

  const text = response.content[0].text.trim();

  try {
    return JSON.parse(text);
  } catch {
    // Tenta extrair JSON caso venha com texto extra
    const match = text.match(/\{[\s\S]*\}/);
    if (match) return JSON.parse(match[0]);
    throw new Error("IA não retornou um JSON válido: " + text);
  }
}
```

### Exemplo de saída para o tema `"animais correndo na floresta com música clássica"`

```json
{
  "title": "Animals Running Through the Forest",
  "duration_seconds": 60,
  "clips": [
    {
      "order": 1,
      "search_query": "deer running through forest sunlight",
      "duration_seconds": 14,
      "color_grade": "warm",
      "description": "Cervo correndo pela floresta com luz solar filtrando pelas árvores"
    },
    {
      "order": 2,
      "search_query": "wolves pack running wild forest",
      "duration_seconds": 12,
      "color_grade": "cool",
      "description": "Alcateia de lobos correndo em grupo pela floresta densa"
    },
    {
      "order": 3,
      "search_query": "fox running through autumn forest leaves",
      "duration_seconds": 12,
      "color_grade": "warm",
      "description": "Raposa correndo por folhas de outono no chão da floresta"
    },
    {
      "order": 4,
      "search_query": "horses galloping meadow forest edge",
      "duration_seconds": 12,
      "color_grade": "vibrant",
      "description": "Cavalos galopando na beira da floresta com prado aberto"
    },
    {
      "order": 5,
      "search_query": "bear walking forest wilderness wildlife",
      "duration_seconds": 10,
      "color_grade": "neutral",
      "description": "Urso caminhando calmamente pela floresta selvagem"
    }
  ],
  "music": {
    "mood": "classical",
    "search_query": "classical orchestral nature forest",
    "volume": 0.4
  },
  "transitions": "dissolve",
  "aspect_ratio": "16:9",
  "resolution": "1920x1080"
}
```

---

## Etapa 2 — Busca e Download de Vídeos

### `src/search/pexels.js`

```javascript
import axios from "axios";

const BASE_URL = "https://api.pexels.com/videos";

export async function searchPexels(query, perPage = 5) {
  const response = await axios.get(`${BASE_URL}/search`, {
    headers: { Authorization: process.env.PEXELS_API_KEY },
    params: {
      query,
      per_page: perPage,
      orientation: "landscape",
      size: "large",
    },
  });

  return response.data.videos.map((video) => {
    // Pega o arquivo de maior resolução disponível
    const bestFile = video.video_files
      .filter((f) => f.quality === "hd" || f.quality === "sd")
      .sort((a, b) => b.width - a.width)[0];

    return {
      id: video.id,
      url: bestFile?.link,
      width: bestFile?.width,
      height: bestFile?.height,
      duration: video.duration,
      source: "pexels",
    };
  }).filter((v) => v.url);
}
```

### `src/search/pixabay.js`

```javascript
import axios from "axios";

export async function searchPixabay(query, perPage = 5) {
  const response = await axios.get("https://pixabay.com/api/videos/", {
    params: {
      key: process.env.PIXABAY_API_KEY,
      q: encodeURIComponent(query),
      per_page: perPage,
      video_type: "film",
      order: "popular",
    },
  });

  return response.data.hits.map((video) => {
    const best = video.videos.large || video.videos.medium;
    return {
      id: video.id,
      url: best?.url,
      width: best?.width,
      height: best?.height,
      duration: video.duration,
      source: "pixabay",
    };
  }).filter((v) => v.url);
}
```

### `src/search/downloader.js`

```javascript
import axios from "axios";
import fs from "fs";
import path from "path";
import { execSync } from "child_process";

export async function downloadVideo(url, filename) {
  const tmpDir = path.resolve("tmp");
  if (!fs.existsSync(tmpDir)) fs.mkdirSync(tmpDir, { recursive: true });

  const outputPath = path.join(tmpDir, filename);

  // Se já existe, reutiliza
  if (fs.existsSync(outputPath)) return outputPath;

  const response = await axios.get(url, { responseType: "stream" });
  const writer = fs.createWriteStream(outputPath);

  await new Promise((resolve, reject) => {
    response.data.pipe(writer);
    writer.on("finish", resolve);
    writer.on("error", reject);
  });

  return outputPath;
}

// Usar yt-dlp para baixar do YouTube (requer yt-dlp instalado no sistema)
export function downloadFromYouTube(url, filename) {
  const outputPath = path.join("tmp", filename);
  execSync(
    `yt-dlp -f "bestvideo[ext=mp4][height<=1080]+bestaudio[ext=m4a]/best[ext=mp4]" ` +
    `-o "${outputPath}" "${url}"`,
    { stdio: "inherit" }
  );
  return outputPath;
}
```

### Estratégia de busca: Pexels → Pixabay → fallback

A recomendação é tentar o Pexels primeiro (qualidade superior), e se não encontrar resultados relevantes, tentar o Pixabay. Baixar sempre 2–3 vídeos por query e usar o primeiro como principal.

---

## Etapa 3 — Edição Automática dos Clipes

Esta é a etapa mais técnica. Usamos o `fluent-ffmpeg` como wrapper do FFmpeg.

### `src/editor/trimmer.js`

```javascript
import ffmpeg from "fluent-ffmpeg";
import path from "path";

/**
 * Corta um vídeo no tempo desejado, iniciando do ponto mais interessante.
 * Para simplicidade, começa sempre do segundo 2 (evita fade in de câmera).
 */
export function trimClip(inputPath, durationSeconds, outputFilename) {
  return new Promise((resolve, reject) => {
    const outputPath = path.join("tmp", outputFilename);

    ffmpeg(inputPath)
      .setStartTime(2)                       // Pula os primeiros 2 segundos
      .setDuration(durationSeconds)          // Duração definida pela IA
      .videoCodec("libx264")
      .audioCodec("aac")
      .outputOptions([
        "-vf scale=1920:1080:force_original_aspect_ratio=decrease,pad=1920:1080:(ow-iw)/2:(oh-ih)/2", // Centraliza e padroniza resolução
        "-r 30",                             // 30fps
        "-crf 23",                           // Qualidade balanceada
        "-preset fast",
      ])
      .output(outputPath)
      .on("end", () => resolve(outputPath))
      .on("error", reject)
      .run();
  });
}
```

### `src/editor/grader.js`

Color grading define o "clima" visual do clipe. Cada preset manipula brilho, contraste, saturação e curvas de cor.

```javascript
import ffmpeg from "fluent-ffmpeg";
import path from "path";

const GRADES = {
  warm: "curves=r='0/0 0.5/0.6 1/1':g='0/0 0.5/0.48 1/0.9':b='0/0 0.5/0.4 1/0.8',eq=brightness=0.05:saturation=1.2",
  cool:  "curves=r='0/0 0.5/0.4 1/0.85':g='0/0 0.5/0.5 1/0.9':b='0/0 0.5/0.6 1/1.1',eq=brightness=0.02:saturation=0.9",
  vibrant: "eq=saturation=1.5:contrast=1.1:brightness=0.03",
  neutral: "eq=saturation=1.0:contrast=1.0:brightness=0.0",
};

export function applyColorGrade(inputPath, grade, outputFilename) {
  return new Promise((resolve, reject) => {
    const outputPath = path.join("tmp", outputFilename);
    const filter = GRADES[grade] || GRADES.neutral;

    ffmpeg(inputPath)
      .videoFilters(filter)
      .videoCodec("libx264")
      .audioCodec("copy")
      .outputOptions(["-crf 23", "-preset fast"])
      .output(outputPath)
      .on("end", () => resolve(outputPath))
      .on("error", reject)
      .run();
  });
}
```

### `src/editor/transitions.js`

```javascript
import ffmpeg from "fluent-ffmpeg";
import fs from "fs";
import path from "path";

/**
 * Junta múltiplos clipes com transição de dissolve (cross-fade).
 * Cada transição dura 1 segundo.
 */
export function joinWithTransitions(clipPaths, transitionType, outputPath) {
  return new Promise((resolve, reject) => {
    if (transitionType === "dissolve") {
      return joinWithDissolve(clipPaths, outputPath, resolve, reject);
    }
    // Fallback: concatenação direta com fade
    return joinWithFade(clipPaths, outputPath, resolve, reject);
  });
}

function joinWithFade(clipPaths, outputPath, resolve, reject) {
  // Cria arquivo de lista para concatenação
  const listFile = path.join("tmp", "concat_list.txt");
  const content = clipPaths.map((p) => `file '${path.resolve(p)}'`).join("\n");
  fs.writeFileSync(listFile, content);

  ffmpeg()
    .input(listFile)
    .inputOptions(["-f concat", "-safe 0"])
    .videoFilters([
      // Aplica fade out no final de cada clipe e fade in no início do próximo
      "fade=t=in:st=0:d=0.5",
      "fade=t=out:st=0:d=0.5",
    ])
    .videoCodec("libx264")
    .audioCodec("aac")
    .outputOptions(["-crf 22", "-preset fast"])
    .output(outputPath)
    .on("end", () => resolve(outputPath))
    .on("error", reject)
    .run();
}

function joinWithDissolve(clipPaths, outputPath, resolve, reject) {
  // Dissolve real entre clipes usando xfade filter do FFmpeg
  let command = ffmpeg();

  clipPaths.forEach((p) => command.input(p));

  const filterParts = [];
  let prevOut = "0:v";
  const transitionDuration = 1; // 1 segundo de transição

  for (let i = 1; i < clipPaths.length; i++) {
    const outLabel = i === clipPaths.length - 1 ? "vout" : `v${i}`;
    filterParts.push(
      `[${prevOut}][${i}:v]xfade=transition=dissolve:duration=${transitionDuration}:offset=${i * 12 - transitionDuration}[${outLabel}]`
    );
    prevOut = outLabel;
  }

  command
    .complexFilter(filterParts.join(";"))
    .outputOptions(["-map [vout]", "-crf 22", "-preset fast"])
    .output(outputPath)
    .on("end", () => resolve(outputPath))
    .on("error", reject)
    .run();
}
```

---

## Etapa 4 — Busca e Mix de Música

### `src/music/finder.js`

```javascript
import axios from "axios";
import fs from "fs";
import path from "path";

/**
 * Busca músicas no Pixabay Music (gratuito, sem copyright).
 * Categoria pode ser: classical, ambient, cinematic, jazz, electronic, etc.
 */
export async function findMusic(mood, searchQuery) {
  const categoryMap = {
    classical: "classical",
    ambient: "ambient",
    calm: "ambient",
    dramatic: "cinematic",
    epic: "cinematic",
    upbeat: "electronic",
  };

  const category = categoryMap[mood] || "ambient";

  const response = await axios.get("https://pixabay.com/api/", {
    params: {
      key: process.env.PIXABAY_API_KEY,
      q: encodeURIComponent(searchQuery),
      category,
      media_type: "music",
      per_page: 5,
    },
  });

  const tracks = response.data.hits;
  if (!tracks || tracks.length === 0) {
    throw new Error(`Nenhuma música encontrada para mood: ${mood}`);
  }

  // Retorna a mais popular
  return {
    id: tracks[0].id,
    url: tracks[0].audio,
    title: tracks[0].tags,
    duration: tracks[0].duration,
  };
}

export async function downloadMusic(url, filename) {
  const tmpDir = path.resolve("tmp");
  if (!fs.existsSync(tmpDir)) fs.mkdirSync(tmpDir, { recursive: true });

  const outputPath = path.join(tmpDir, filename);
  if (fs.existsSync(outputPath)) return outputPath;

  const response = await axios.get(url, { responseType: "stream" });
  const writer = fs.createWriteStream(outputPath);

  await new Promise((resolve, reject) => {
    response.data.pipe(writer);
    writer.on("finish", resolve);
    writer.on("error", reject);
  });

  return outputPath;
}
```

### `src/music/mixer.js`

```javascript
import ffmpeg from "fluent-ffmpeg";

/**
 * Mistura áudio de fundo com o vídeo.
 * - Reduz o volume da música conforme definido pela IA
 * - Aplica fade out nos últimos 3 segundos
 * - Remove o áudio original do vídeo (opcional)
 */
export function mixAudio(videoPath, musicPath, volume, outputPath) {
  return new Promise((resolve, reject) => {
    ffmpeg()
      .input(videoPath)
      .input(musicPath)
      .complexFilter([
        // Ajusta volume da música e aplica fade out nos últimos 3s
        `[1:a]volume=${volume},afade=t=out:st=-3:d=3[music]`,
        // Fade in no vídeo nos primeiros 0.5s
        `[0:a]afade=t=in:st=0:d=0.5[video_audio]`,
        // Mistura os dois áudios
        `[video_audio][music]amix=inputs=2:duration=first[aout]`,
      ])
      .outputOptions([
        "-map 0:v",
        "-map [aout]",
        "-c:v copy",
        "-c:a aac",
        "-shortest",
      ])
      .output(outputPath)
      .on("end", () => resolve(outputPath))
      .on("error", reject)
      .run();
  });
}
```

---

## Etapa 5 — Renderização Final

### `src/renderer/render.js`

```javascript
import path from "path";
import fs from "fs";
import { trimClip } from "../editor/trimmer.js";
import { applyColorGrade } from "../editor/grader.js";
import { joinWithTransitions } from "../editor/transitions.js";
import { mixAudio } from "../music/mixer.js";
import { searchPexels } from "../search/pexels.js";
import { searchPixabay } from "../search/pixabay.js";
import { downloadVideo } from "../search/downloader.js";
import { findMusic, downloadMusic } from "../music/finder.js";
import { log } from "../utils/logger.js";

export async function renderVideo(plan) {
  const processedClips = [];

  log.info(`Iniciando renderização: "${plan.title}"`);

  // ─── Processa cada clipe ───────────────────────────────────────────────────
  for (const clip of plan.clips) {
    log.step(`Clipe ${clip.order}: ${clip.search_query}`);

    // 1. Busca vídeo
    let videos = await searchPexels(clip.search_query, 3);
    if (!videos.length) {
      log.warn("Pexels sem resultado, tentando Pixabay...");
      videos = await searchPixabay(clip.search_query, 3);
    }
    if (!videos.length) throw new Error(`Sem vídeos para: ${clip.search_query}`);

    // 2. Download
    const rawPath = await downloadVideo(
      videos[0].url,
      `raw_${clip.order}.mp4`
    );
    log.ok(`Download OK: ${rawPath}`);

    // 3. Corte
    const trimmedPath = await trimClip(
      rawPath,
      clip.duration_seconds,
      `trimmed_${clip.order}.mp4`
    );
    log.ok(`Corte OK: ${clip.duration_seconds}s`);

    // 4. Color grade
    const gradedPath = await applyColorGrade(
      trimmedPath,
      clip.color_grade,
      `graded_${clip.order}.mp4`
    );
    log.ok(`Color grade OK: ${clip.color_grade}`);

    processedClips.push(gradedPath);
  }

  // ─── Junta os clipes com transições ───────────────────────────────────────
  log.step("Juntando clipes com transições...");
  const outputDir = path.resolve("output");
  if (!fs.existsSync(outputDir)) fs.mkdirSync(outputDir, { recursive: true });

  const joinedPath = path.join("tmp", "joined.mp4");
  await joinWithTransitions(processedClips, plan.transitions, joinedPath);
  log.ok("Clipes unidos!");

  // ─── Busca e download da música ───────────────────────────────────────────
  log.step(`Buscando música: ${plan.music.mood} / "${plan.music.search_query}"`);
  const track = await findMusic(plan.music.mood, plan.music.search_query);
  const musicPath = await downloadMusic(track.url, "music.mp3");
  log.ok(`Música baixada: ${track.title}`);

  // ─── Mix final ────────────────────────────────────────────────────────────
  log.step("Mixando áudio...");
  const safeTitle = plan.title.replace(/[^a-z0-9]/gi, "_").toLowerCase();
  const finalPath = path.join(outputDir, `${safeTitle}.mp4`);
  await mixAudio(joinedPath, musicPath, plan.music.volume, finalPath);

  log.success(`\n✅ Vídeo gerado com sucesso!\n📁 ${finalPath}\n`);
  return finalPath;
}
```

---

## CLI — Ponto de Entrada

### `index.js`

```javascript
import { program } from "commander";
import "dotenv/config";
import { planVideo } from "./src/ai/planner.js";
import { renderVideo } from "./src/renderer/render.js";
import { cleanup } from "./src/utils/cleanup.js";
import { log } from "./src/utils/logger.js";

program
  .name("video-automator")
  .description("Gera vídeos automaticamente a partir de um tema")
  .version("1.0.0");

program
  .command("generate <theme>")
  .description("Gera um vídeo com base no tema informado")
  .option("--no-cleanup", "Mantém arquivos temporários após renderização")
  .action(async (theme, options) => {
    try {
      log.info(`🎬 Tema: "${theme}"\n`);

      log.step("Planejando vídeo com IA...");
      const plan = await planVideo(theme);
      log.ok(`Plano criado: ${plan.clips.length} clipes, ${plan.duration_seconds}s`);
      log.info(JSON.stringify(plan, null, 2));

      const outputPath = await renderVideo(plan);

      if (options.cleanup) {
        log.step("Limpando arquivos temporários...");
        cleanup();
      }

      log.success(`\n🎉 Concluído! Vídeo em: ${outputPath}`);
    } catch (err) {
      log.error(`Erro: ${err.message}`);
      process.exit(1);
    }
  });

program.parse();
```

### Uso via terminal

```bash
# Instalar dependências
npm install

# Gerar vídeo
node index.js generate "animais correndo na floresta com música clássica"

# Manter arquivos temporários para debug
node index.js generate "pôr do sol na praia" --no-cleanup

# Outros exemplos de temas
node index.js generate "cidade à noite com jazz"
node index.js generate "montanhas cobertas de neve com música épica"
node index.js generate "crianças brincando no parque ao pôr do sol"
```

---

## Variáveis de Ambiente

Crie um arquivo `.env` na raiz do projeto:

```env
# Claude API (Anthropic)
ANTHROPIC_API_KEY=sk-ant-...

# Pexels (https://www.pexels.com/api/)
PEXELS_API_KEY=...

# Pixabay (https://pixabay.com/api/docs/)
PIXABAY_API_KEY=...
```

Arquivo `.env.example` para commitar:

```env
ANTHROPIC_API_KEY=
PEXELS_API_KEY=
PIXABAY_API_KEY=
```

---

## Dependências

### `package.json`

```json
{
  "name": "video-automator",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node index.js",
    "generate": "node index.js generate"
  },
  "dependencies": {
    "@anthropic-ai/sdk": "^0.27.0",
    "axios": "^1.7.0",
    "commander": "^12.0.0",
    "dotenv": "^16.4.0",
    "fluent-ffmpeg": "^2.1.3"
  },
  "devDependencies": {
    "@types/fluent-ffmpeg": "^2.1.27"
  }
}
```

### Instalação das dependências de sistema

```bash
# macOS
brew install ffmpeg yt-dlp

# Ubuntu / Debian
sudo apt install ffmpeg
pip install yt-dlp

# Windows (via Chocolatey)
choco install ffmpeg yt-dlp

# Verificar instalação
ffmpeg -version
yt-dlp --version
```

### Instalar dependências Node

```bash
npm install
```

---

## Roadmap de Evolução

### Versão 1.0 — MVP (base deste guia)
- [x] Parsing do tema com IA
- [x] Busca no Pexels e Pixabay
- [x] Corte, color grade e transições via FFmpeg
- [x] Mix de música
- [x] CLI funcional

### Versão 1.1 — Qualidade
- [ ] Adicionar legendas/subtítulos geradas pela IA
- [ ] Texto animado no início (título do vídeo)
- [ ] Detecção automática do melhor trecho do clipe (cenas com mais movimento)
- [ ] Score de relevância para escolher o melhor vídeo entre os resultados

### Versão 1.2 — Fontes de conteúdo
- [ ] Suporte a Unsplash para imagens estáticas com Ken Burns effect
- [ ] Integração com Coverr (banco de vídeos cinematic)
- [ ] Suporte a YouTube via yt-dlp com validação de licença

### Versão 2.0 — Interface Web
- [ ] API REST com Express.js
- [ ] Frontend simples (Next.js) para gerar vídeos pelo browser
- [ ] Fila de jobs com Bull/BullMQ para processamento em background
- [ ] Preview em tempo real do progresso

### Versão 2.1 — IA Avançada
- [ ] Análise de frame de cada vídeo com Vision API para escolher o melhor clipe
- [ ] Geração de narração com ElevenLabs
- [ ] Geração de imagens com DALL-E para preencher lacunas de conteúdo
- [ ] Script de narração gerado pela IA

### Versão 3.0 — Produção
- [ ] Deploy em AWS Lambda + S3 para processamento serverless
- [ ] Cache de vídeos baixados para reutilização
- [ ] Rate limiting nas APIs externas
- [ ] Dashboard de jobs e histórico de vídeos gerados

---

## Dicas e Armadilhas Comuns

### FFmpeg na PATH
O `fluent-ffmpeg` precisa do FFmpeg instalado e disponível na PATH do sistema. Se der erro, adicione o caminho manualmente:

```javascript
import ffmpeg from "fluent-ffmpeg";
ffmpeg.setFfmpegPath("/usr/local/bin/ffmpeg");
ffmpeg.setFfprobePath("/usr/local/bin/ffprobe");
```

### Limite de requisições nas APIs
O Pexels free permite 200 requisições/hora. O Pixabay permite 100/hora. Implemente um cache simples salvando os resultados em JSON:

```javascript
import fs from "fs";
const CACHE_FILE = "tmp/search_cache.json";

function getCache() {
  if (!fs.existsSync(CACHE_FILE)) return {};
  return JSON.parse(fs.readFileSync(CACHE_FILE, "utf-8"));
}

function setCache(key, value) {
  const cache = getCache();
  cache[key] = { value, ts: Date.now() };
  fs.writeFileSync(CACHE_FILE, JSON.stringify(cache, null, 2));
}
```

### Vídeos com resolução incompatível
Nem todos os vídeos vêm em 1920x1080. O filtro no `trimmer.js` já lida com isso via `pad` e `scale`, mas atenção: se o vídeo for vertical (celular), o resultado vai ter barras pretas. Filtre vídeos com `width > height` na busca.

### Transições dissolve com duração errada
O filtro `xfade` do FFmpeg calcula o offset com base na duração total dos clipes. Se a duração de um clipe for menor que o offset calculado, o FFmpeg vai dar erro silencioso. Sempre valide as durações reais dos arquivos baixados com `ffprobe` antes de montar as transições.

### Música mais curta que o vídeo
Se a música tiver duração menor que o vídeo, o FFmpeg vai silenciar o restante. Use a flag `-stream_loop -1` para fazer o áudio loopar:

```javascript
ffmpeg()
  .input(musicPath)
  .inputOptions(["-stream_loop -1"]) // Loop infinito
  // ...
```

### Timeout em downloads longos
Use `axios` com timeout configurado para evitar travamentos:

```javascript
const response = await axios.get(url, {
  responseType: "stream",
  timeout: 60000, // 60 segundos
});
```

---

*Documentação gerada para o projeto Video Automator — Node.js + FFmpeg + Claude API*
