# DSR Native — trabalho de desempenho na GPU Mali

Resumo do que foi feito e do que se descobriu até agora, medido num Samsung SM-A075M
(MediaTek MT6789 / Helio G99, Mali-G57, ~7,6 GB de RAM, Android 17). Meta: **30 FPS**.

> O `vk_backend.cpp` não pôde ser compilado fora do GitHub Actions; tudo foi validado pelo
> log e pelas capturas do próprio aparelho. O que não foi medido está marcado como estimativa.

Estado atual (build `mali-13`): **imagem correta**, ~30 FPS em lugar fechado, ~24–26 FPS em área
aberta (Undead Burg / dragão). Falta cortar ~8–10 ms de custo fixo na área aberta.

---

## 1. Ponto de partida

| Situação | Tempo por quadro | FPS |
|---|---|---|
| Normal (sem escala) | ~58 ms, GPU 55–60 ms por lista | ≈17 |
| `viewport_test` (1/4 dos pixels, só diagnóstico) | 42–43 ms, GPU 34–43 ms por lista | ≈23 |

Cortar 75% dos pixels tirou ~16 ms e ainda sobravam ~35–40 ms de **custo fixo** (~1.040 draws,
62 passes, sombra 2048x2048 e ~50 passes pequenos de pós-processamento).

---

## 2. O que foi implementado

### 2.1 `render_scale` (escala de resolução real) — funciona
Opção do `settings.txt` (0,5 a 1,0; 1,0 = comportamento antigo).

- Alvos de tela cheia (1280x720, cor e profundidade, 2D, 1 camada) são **alocados menores**
  (ex.: 960x540 em 0,75; 768x432 em 0,6). Sombra e alvos pequenos de pós-processamento não mudam.
- `Image` ganhou `vk_w/vk_h` (tamanho real no Vulkan); `width/height` continuam os do jogo.
- Viewport e scissor são escalados só no passe escalado (`RenderScaled(extent)`).
- Extents do passe, clears, `CopyImage` e o blit final usam o tamanho real.
- **`gl_FragCoord.xy`** é multiplicado por `1/escala` no tradutor (push constant `push[1]`).
- **Texel fetch (`TLD`)**: coordenadas inteiras em texturas escaladas são multiplicadas pela
  escala, usando uma máscara por draw dos slots de textura escalados (`push[2]`, `push[3]`).

Push constants (fragment): `{ alpha_ref, fragcoord_mul, mascara_texturas_escaladas, fator_escala }`.

**Bug 1 (corrigido antes):** a cena aparecia no canto superior esquerdo e o resto ficava cinza,
porque o `TLD` pedia o pixel 1000 de uma textura de 960. Corrigido com a máscara por slot.

**Bug 2 (corrigido no `mali-13`): scissor escalado duas vezes.** No `mali-12`, o scissor era
limitado ao tamanho **já reduzido** (960x540) e só depois multiplicado pela escala, dando
`720x405`. Efeitos:
- a tela mostrava só os 75% superiores esquerdos da cena (recorte, resto preto);
- a GPU desenhava só 56% dos pixels, então o FPS daquela rodada estava **inflado**;
- a sombra ficava com scissor de área zero (`512,512 0x0`), então o passe da sombra quase não
  desenhava (custo ~1,2 ms falso) e as sombras sumiam.

Correção: limitar o scissor ao tamanho do jogo (1280x720, ou 2048 na sombra) **antes** de escalar,
e usar esse tamanho também como scissor padrão. Como conferir no log (`frame capture`):
`scissor 0,0 960x540 on 960x540` (0,75) ou `scissor 0,0 768x432 on 768x432` (0,6) nos draws de
tela cheia. Se aparecer `720x405`, o build não tem a correção.

### 2.2 `shadow_size` — funciona
Aloca o mapa de sombra 2048x2048 em tamanho menor (256–2047; usado: 1024). Viewport e scissor da
sombra são escalados pela mesma rotina (depende da correção do bug 2).
Custo do passe da sombra nas capturas corretas: 2,3–4,3 ms (inflados pela captura), com 35–115 draws.
Não houve teste A/B limpo contra 2048 depois da correção.

### 2.3 `skip_depth_prepass` — funciona (ganho isolado não medido)
A cena de tela cheia é desenhada duas vezes (passe só de profundidade, depois passe de cor).
No Mali o primeiro quase não ajuda.

- Pula draws que só escrevem profundidade (máscara de cor 0 + depth write ligado), apenas em
  passes cujo alvo de profundidade é um alvo escalado de largura 1280 (a sombra não é tocada).
- No passe de cor seguinte (sem blend, depth test sem escrita, no mesmo quadro) liga a escrita de
  profundidade; `EQUAL` vira `LESS_OR_EQUAL` (o jogo já usa `LEQUAL` nesse passe).
- Confirmado no log: `skip_depth_prepass: skipping depth-only draws of the full-screen pass`, e a
  categoria "depth-only (screen)" some do resumo da captura (antes: 5,5–8,6 ms inflados).
- É heurística. Se algo sumir ou ordenar errado: `skip_depth_prepass = 0`.
- Ganho real: estimativa ~3 ms por quadro. Não houve A/B (mesma cena, ligado/desligado).

O erro da rodada em que "não teve efeito" foi simples: a opção estava desligada
(`skip_depth_prepass off` na linha `settings:` do log).

### 2.4 `skip_shaders` — experimento (build `mali-14`, não compilado nem testado)
Lista de hashes de shader de fragmento (8 primeiros dígitos hex, separados por vírgula, sem espaços)
cujos draws são descartados. Serve para descobrir, no aparelho e sem recompilar, quais passes de
pós-processamento valem o custo. O alvo do passe fica com o que já tinha (limpo ou do quadro
anterior), então podem aparecer artefatos, tela escura ou cores estranhas: é o resultado do teste.
Vazio = imagem normal. O log mostra `settings: skip_shaders: <hash>` e
`skip_shaders: dropping draws of fragment shader ...` quando pega. Tabela dos passes e ordem de
teste em `SKIP_SHADERS.md`.

---

## 3. Resultados medidos

Média de quadro em área aberta = média das janelas de 10 s do log (`frame times`).

| Build / teste | Resultado | Observação |
|---|---|---|
| Sem escala, área aberta (relato) | 16–18 FPS | ponto de partida |
| `0.75`, prisão (antigo) | ~33,5 ms, 30 FPS | área fechada |
| `mali-12`, `0.75` + `shadow 1024`, skip ligado | ~41,8 ms | **inválido**: scissor duplo, imagem cortada |
| `mali-13`, `0.75` + `shadow 1024` + skip, área aberta | ~43,3 ms (≈23 FPS), GPU 36–44 ms por lista | imagem correta |
| `mali-13`, mesmo, lugar fechado | 26–27 FPS; 30 FPS só em trechos leves (GPU 5–13 ms por lista) | |
| `mali-13`, `0.6` + `shadow 1024` + skip, área aberta | ~42,2 ms (≈24 FPS), GPU 34–42 ms por lista | relato: 30 e 26 fechado, 26 e 24 aberto |

Capturas da área aberta (inflam ~2x; vale a proporção):

| Grupo | `0.75` (cap. 4) | `0.6` (cap. 4) |
|---|---|---|
| Cor opaca (tela) | 33,4 ms | 30,0 ms |
| Pós-processamento (31 passes) | 25,8 ms | 26,0 ms |
| Cópias e clears | 11,5 ms | 8,7 ms |
| Blended | 8,8 ms | 6,5 ms |
| Sombra | 2,9 ms | 2,6 ms |
| **Total** | **82,4 ms** | **73,8 ms** |

Leitura: **baixar de 0,75 para 0,6 quase não mexeu no tempo real** (~1 ms no quadro médio).
A área aberta já não é limitada por pixel; o limite é o custo fixo (passes e draws).
Voltar para `0.75` (mais nítido) é razoável.

---

## 4. Onde está o custo agora (área aberta)

- ~1.100–1.300 draws e ~45 passes por quadro; ~98 barreiras de imagem.
- O passe principal de cor (~365 draws) é sozinho ~28% da captura (~23 ms inflados).
- ~31 passes pequenos de pós-processamento ≈ 31% da captura, sem um passe dominante
  (maiores: ~5,7, ~3,3 e ~3,2 ms inflados). A captura exagera nos passes pequenos.
- Quebras de passe por leitura de textura (janela de 10 s, ~220 quadros): 320x180: 1.656;
  640x360: 1.108; 1280x720: 894 (~4 por quadro); 160x90: 828; 16x16: 414;
  profundidade 640x360, 320x180 e 2048x2048: 207 cada.
- GPU ocupada ~9–10 s a cada 10 s: o limite é a GPU. A thread de render quase não espera pelo gravador.
- Hitches (quadros > 67 ms): 10–40 por janela de 10 s em área aberta.
- ~1.100 alvos pequenos escritos de volta na memória do jogo por 10 s (leituras de exposição etc.).

---

## 5. O que ainda não se sabe

- Quanto o `skip_depth_prepass` rende de fato (falta A/B ligado/desligado na mesma cena).
- Quanto o `shadow_size = 1024` rende contra 2048 com o scissor correto.
- Quais dos ~31 passes de pós-processamento são opcionais (brilho, profundidade de campo etc.).
- A contagem de "samples passed" por passe não bateu com o esperado e não foi usada.
- O `TLD` ainda pode falhar em algum shader específico; leituras de volta para a CPU de alvos
  escalados não foram tratadas.
- A origem dos hitches de centenas de ms (ex.: 500–1.100 ms de pior quadro em algumas janelas).

---

## 6. Como usar e testar

`Download/DSRNative/settings.txt` (lido só ao abrir o jogo; feche-o por completo antes):

```
render_scale = 0.75        # 0.6 não rendeu ganho claro; 1.0 desliga
shadow_size = 1024         # 0 = como o jogo pede
skip_depth_prepass = 1     # experimental; 0 desliga
profile = 0                # 1 só para diagnóstico
texture_drop = 2           # 1 se as texturas ficarem ruins demais
skip_shaders =             # experimento; vazio = normal (ver SKIP_SHADERS.md)
auto_capture = 0           # L3 + R3 juntos na tela sempre salva uma captura
```

(O comentário do `settings.txt` gerado diz "skip_depth_prepass: no gain measured; keep 0";
está desatualizado, a opção funciona.)

Teste no ponto aberto (dragão): rode ~30 s, aperte L3 + R3 uma vez, feche e leve o `dsr_*.txt`
(os `.bin` de ~30 MB só servem para ver imagens dos passes).

No log procure:
- `settings: render_scale ..., shadow_size ..., skip_depth_prepass on/off` (opções lidas);
- `render_scale: 1280x720 target ... allocated at ...` e `shadow_size: ... allocated at ...`;
- `scissor 0,0 WxH on WxH` nos draws de tela cheia (deve ser igual ao alvo, sem `720x405`);
- `skip_depth_prepass: skipping depth-only draws...`;
- `frame times: ... ms average` e `GPU busy ... ms per list`;
- `capture N SUMMARY` para a divisão do tempo por grupo.

---

## 7. Próximos passos (em ordem)

1. Fixar `render_scale = 0.75` (ou testar 0,85 se sobrar folga) e medir A/B do `skip_depth_prepass`.
2. Testar o `skip_shaders` (ver `SKIP_SHADERS.md`), um grupo por vez, e conferir imagem e FPS.
3. Reduzir as quebras de passe para leitura de textura, começando pela de 1280x720 (~4 por quadro).
4. Atacar a contagem de draws do passe principal (~365) e os hitches de centenas de ms.
