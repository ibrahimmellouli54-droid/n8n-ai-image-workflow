# 🎨 AI Image Generation Workflow — n8n + Ollama + Pollinations + SD Forge

> Workflow n8n de génération d'images par IA avec optimisation automatique de prompts, critique multi-modèle et boucle d'amélioration itérative.

---

## 📋 Table des matières

- [Vue d'ensemble](#vue-densemble)
- [Architecture du workflow](#architecture-du-workflow)
- [Prérequis](#prérequis)
- [Installation étape par étape](#installation-étape-par-étape)
- [Configuration des credentials](#configuration-des-credentials)
- [Import du workflow](#import-du-workflow)
- [Utilisation](#utilisation)
- [Dépannage](#dépannage)
- [Liens utiles](#liens-utiles)

---

## Vue d'ensemble

Ce workflow automatise la génération d'images de haute qualité via une pipeline complète :

1. **Entrée** : l'utilisateur envoie un prompt texte (et optionnellement une image de référence) via le chat n8n
2. **Analyse** : si une image est fournie, Qwen2.5-VL l'analyse et fusionne son contenu avec le prompt
3. **Génération initiale** : 3 images sont générées en parallèle via Pollinations.ai (modèles SDXL, Flux, Turbo)
4. **Critique automatique** : chaque image est notée par Qwen2.5-VL sur 3 critères (fidélité, esthétique, technique)
5. **Optimisation itérative** : Qwen3 améliore les prompts jusqu'à obtenir un score > 9/10 (max 3 boucles)
6. **Rendu final** : DeepSeek-R1 génère le prompt ultime → image finale via Stable Diffusion Forge (img2img)

---

## Architecture du workflow

```
[Chat Trigger]
      │
      ▼
[Détecteur] ── image? ──YES──> [base64] → [Qwen2.5 analyse] → [Fusion]
      │                                                              │
      NO                                                             │
      ▼                                                             ▼
[Prompt Engineer Sans image] ←────────────────────────────────────┘
      │
      ▼
[URL Maker] → [Gen1 (SDXL)] ┐
              [Gen2 (Flux)]  ├─> [Base64] → [Prompt Critique] → [Critique Ollama] → [Extract Score]
              [Gen3 (Turbo)] ┘
                                                   │
                                                   ▼
                                           [Merge + Comparaison]
                                                   │
                                    score > 9 ou loop >= 3?
                                          │              │
                                         YES             NO
                                          │              │
                                          │         [Prompt Optimizer (Qwen3)]
                                          │              │
                                          │         [Gen4/5/6 + Critique] ──> boucle
                                          │
                                          ▼
                                    [AI Agent (DeepSeek-R1)]
                                          │
                                          ▼
                              [SD Forge img2img /sdapi/v1/img2img]
                                          │
                                          ▼
                                    [Image finale]
```

---

## Prérequis

| Composant | Version minimale | Rôle |
|---|---|---|
| **n8n** | ≥ 1.40.0 | Orchestration du workflow |
| **Docker** | ≥ 24.0 | Conteneurisation n8n |
| **Ollama** | ≥ 0.3.0 | Serveur LLM local |
| **Stable Diffusion WebUI Forge** | latest | Génération finale img2img |
| **GPU** | 8 GB VRAM (recommandé) | Inférence Ollama + SD Forge |

---

## Installation étape par étape

### Étape 1 — Installer Ollama

**Linux / macOS :**
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

**Windows :** télécharger l'installateur sur [https://ollama.com/download](https://ollama.com/download)

Vérifier que le service tourne :
```bash
ollama --version
```

#### Télécharger les modèles requis

```bash
# Modèle de prompt engineering et optimisation (8B)
ollama pull qwen3:latest

# Modèle vision pour l'analyse d'images et la critique (7B)
ollama pull qwen2.5vl:7b

# Modèle de raisonnement pour le prompt final (8B)
ollama pull deepseek-r1:latest
```

> ⚠️ Les 3 modèles représentent environ **15–20 GB** d'espace disque total.

Vérifier les modèles installés :
```bash
ollama list
```

---

### Étape 2 — Installer n8n via Docker

```bash
docker run -d \
  --name n8n \
  --restart unless-stopped \
  -p 5678:5678 \
  -e N8N_SECURE_COOKIE=false \
  -v n8n_data:/home/node/.n8n \
  --add-host=host.docker.internal:host-gateway \
  docker.n8n.io/n8nio/n8n
```

> 🔑 **Important** : le flag `--add-host=host.docker.internal:host-gateway` est indispensable. Il permet au conteneur n8n de communiquer avec Ollama et SD Forge qui tournent sur la machine hôte.

Accéder à n8n : [http://localhost:5678](http://localhost:5678)

---

### Étape 3 — Installer Stable Diffusion WebUI Forge

**Cloner le dépôt :**
```bash
git clone https://github.com/lllyasviel/stable-diffusion-webui-forge.git
cd stable-diffusion-webui-forge
```

**Lancer Forge avec l'API activée :**
```bash
# Linux / macOS
./webui.sh --api --listen

# Windows
webui-user.bat
# Ajouter --api dans COMMANDLINE_ARGS dans webui-user.bat
```

Vérifier que l'API est accessible :
```
http://localhost:7860/sdapi/v1/sd-models
```

#### Télécharger un modèle pour SD Forge

Placer un modèle `.safetensors` dans le dossier `models/Stable-diffusion/`.

Recommandé pour le rendu réaliste :
- **Realistic Vision V6** : [https://civitai.com/models/4201](https://civitai.com/models/4201)

---

### Étape 4 — Vérifier la connectivité

Depuis le terminal, tester que n8n peut joindre Ollama et SD Forge :

```bash
# Test Ollama
curl http://localhost:11434/api/tags

# Test SD Forge API
curl http://localhost:7860/sdapi/v1/sd-models
```

Si vous utilisez Docker sur Linux, tester avec `host.docker.internal` :
```bash
docker exec n8n curl http://host.docker.internal:11434/api/tags
docker exec n8n curl http://host.docker.internal:7860/sdapi/v1/sd-models
```

---

## Configuration des credentials

### Credential Ollama dans n8n

1. Dans n8n, aller dans **Settings → Credentials → New Credential**
2. Chercher **Ollama**
3. Renseigner :
   - **Base URL** : `http://host.docker.internal:11434`
4. Sauvegarder sous le nom : `Ollama account`

Ce credential est utilisé par les nœuds : `Qwen3 I`, `Qwen3 II`, `Qwen3 III`, `Ollama Chat Model`.

> ℹ️ Aucun credential n'est nécessaire pour **Pollinations.ai** — l'API est publique et gratuite.  
> ℹ️ Aucun credential n'est nécessaire pour **SD Forge** — l'API locale ne requiert pas d'authentification par défaut.

---

## Import du workflow

1. Ouvrir n8n : [http://localhost:5678](http://localhost:5678)
2. Aller dans **Workflows** → bouton **+** → **Import from file**
3. Sélectionner le fichier `My_workflow__29_.json`
4. Cliquer sur **Import**
5. Vérifier que les credentials Ollama sont bien liés aux nœuds concernés
6. Sauvegarder le workflow (Ctrl+S)
7. Activer le workflow avec le toggle en haut à droite

---

## Utilisation

### Lancer le chat

1. Ouvrir le nœud **Tchat** (Chat Trigger) dans l'éditeur
2. Cliquer sur **Open Chat** pour ouvrir l'interface de chat
3. Ou accéder via le lien de production du workflow

### Mode texte seul

Taper simplement votre description dans le chat :
```
Un renard roux assis sur un rocher au coucher du soleil, style photoréaliste
```

### Mode texte + image de référence

Attacher une image dans le chat (le workflow détecte automatiquement sa présence) puis décrire la scène souhaitée :
```
[image attachée : photo d'un chat roux]
Place ce chat dans une forêt enchantée au crépuscule
```

### Ce qui se passe ensuite

- Le workflow génère **3 images en parallèle** avec différents modèles Pollinations
- Les images sont **notées automatiquement** par l'IA (Fidelity / Aesthetic / Technical)
- Si le meilleur score est < 9/10, les prompts sont **optimisés** et de nouvelles images générées (max 3 boucles)
- La meilleure image passe par **SD Forge img2img** pour le rendu final
- L'image finale est retournée dans le chat

---

## Dépannage

| Problème | Cause probable | Solution |
|---|---|---|
| `Connection refused` sur Ollama | Ollama n'est pas démarré | `ollama serve` |
| `host.docker.internal` non résolu | Docker Linux sans flag | Ajouter `--add-host=host.docker.internal:host-gateway` au `docker run` |
| SD Forge inaccessible | Forge lancé sans `--api` | Relancer avec `--api --listen` |
| Modèle Ollama introuvable | Modèle non téléchargé | `ollama pull <nom_du_modèle>` |
| Timeout sur la génération | Modèles trop lourds pour le GPU | Réduire `max_tokens` ou utiliser des modèles plus légers |
| Credential Ollama non lié | Import partiel | Ré-assigner manuellement le credential dans chaque nœud Ollama |

---

## Liens utiles

| Ressource | URL |
|---|---|
| n8n Documentation | [https://docs.n8n.io](https://docs.n8n.io) |
| n8n Docker Hub | [https://hub.docker.com/r/n8nio/n8n](https://hub.docker.com/r/n8nio/n8n) |
| Ollama — Site officiel | [https://ollama.com](https://ollama.com) |
| Ollama — Bibliothèque de modèles | [https://ollama.com/library](https://ollama.com/library) |
| Qwen3 sur Ollama | [https://ollama.com/library/qwen3](https://ollama.com/library/qwen3) |
| Qwen2.5-VL sur Ollama | [https://ollama.com/library/qwen2.5vl](https://ollama.com/library/qwen2.5vl) |
| DeepSeek-R1 sur Ollama | [https://ollama.com/library/deepseek-r1](https://ollama.com/library/deepseek-r1) |
| Pollinations.ai API | [https://pollinations.ai](https://pollinations.ai) |
| SD WebUI Forge GitHub | [https://github.com/lllyasviel/stable-diffusion-webui-forge](https://github.com/lllyasviel/stable-diffusion-webui-forge) |
| Realistic Vision (CivitAI) | [https://civitai.com/models/4201](https://civitai.com/models/4201) |

---

## Modèles Ollama utilisés

| Nœud n8n | Modèle | Usage |
|---|---|---|
| Qwen3 I, II, III | `qwen3:latest` | Prompt engineering, Fusion, Optimisation |
| Ollama Chat Model | `deepseek-r1:latest` | Génération du prompt final ultime |
| Critiques 1/2/3 | `qwen2.5vl:7b` | Analyse visuelle + notation des images |

---

*Workflow créé avec [n8n](https://n8n.io) — plateforme d'automatisation open-source.*
