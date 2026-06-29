# Gauge — redesign visual (fork do Siphon)

**Data:** 2026-06-29
**Autor:** Arthur Galvão (+ Claude)
**Status:** aprovado para implementação

---

## Visão geral

`Gauge` é um fork pessoal do [Siphon](https://github.com/appariciojunior/siphonClaudeUsage), um app de barra de menu para macOS que mostra o uso do Claude (cota de sessão, cota semanal e custo de tokens). Este projeto **não muda a engenharia nem as funcionalidades** do Siphon: é um **redesign visual** que aplica o design system do Revelatio ao app, mais a troca de nome e ícone.

A entrega final é um **`.dmg` standalone**, instalável, compilado a partir do código com a marca e o visual novos.

### O que muda
- Nome do app: **Siphon → Gauge**
- Identidade visual: paleta, tipografia, superfícies, cantos, ícone (design system Revelatio)
- Cor das barras de uso: semáforo dessaturado para caber no preto

### O que NÃO muda (não-objetivos)
- Lógica de OAuth/PKCE, refresh de token, leitura de quotas (`QuotaService`, `OAuthService`)
- Leitura dos arquivos locais de custo (`LocalDataService`)
- Estrutura de dados, modelos, fluxo de atualização (refresh a cada 30s)
- Endpoints, client ID, headers
- Nenhuma feature nova (sem histórico, gráficos, alertas, multi-conta)

---

## Design system (tokens Revelatio)

Referência: `Design Process` / `revelatio-comercial-app/DESIGN.md`. Material 3 dark + HIG macOS, grade de 8px, mono, hierarquia por opacidade.

### Cores
```
bg            #0a0a0c     canvas (fundo da janela)
surface       #161719     painel do popover (base de elevação)
surface-2     #1f2023     cards / campos
surface-3     #26272b     seleção ativa / raised
outline       rgba(255,255,255,.12)   hairline forte
outline-2     rgba(255,255,255,.07)   hairline padrão (divisórias)
t1            #f4f4f6     títulos e números
t2            rgba(255,255,255,.66)   corpo
t3            rgba(255,255,255,.46)   labels e legendas
t4            rgba(255,255,255,.30)   numerais fracos, estados vazios
```

### Semáforo de uso (exceção controlada à regra mono)
Versões dessaturadas para o fundo preto. Única cor permitida no app; usada só nas barras/números de cota.
```
verde    #5fb07a   0–30%
laranja  #d89a5b   31–75%
vermelho #cc6b61   76–100%
```

### Tipografia
SF Pro via stack de sistema (`-apple-system, BlinkMacSystemFont, "SF Pro Text", "SF Pro Display"`). **Remove a fonte Inter** empacotada e o registro de fontes. Hierarquia por peso (400/600) e opacidade, com diferenças de tamanho modestas.

### Cantos (rounded)
```
sm  10px   chips, botões pequenos
md  14px   ícone do app (tile), campos
lg  16px   popover (shell da janela)
```

### Sombra
Uma única sombra sutil e funcional no painel do popover. Nada de glow, gradiente colorido, vidro fosco ou textura.

---

## Decisões travadas (do brainstorming visual)

| Item | Decisão |
|---|---|
| Nome | **Gauge** |
| Cor das barras | Semáforo dessaturado (verde/laranja/vermelho acima) |
| Ícone | **Anel de 3/4** (medidor de progresso) |
| Fonte | SF Pro (sistema), remove Inter |
| Superfície | Sólida (`surface`) + sombra sutil, remove `.ultraThinMaterial` |
| Cantos | Arredondados 16px (remove o `SquareWindowFix`) |
| Marca | Sem logo. Projeto pessoal; só o glifo do medidor |

---

## Mudanças componente a componente

Arquivo principal: `Siphon/Views/UsageView.swift`. Também: `SiphonApp.swift` (label da barra de menu, registro de fonte) e `Views/PhosphorDrop.swift` (ícone).

### Fonte
- Remover o `extension Font { inter(...) }` e os `.font(.inter(...))`; trocar por `.system(size:weight:)` (SF Pro).
- `SiphonApp.swift`: remover `registerBundledFonts()` e a pasta `Fonts/`.
- `LabelRenderer` (barra de menu): trocar `NSFont(name: "Inter-SemiBold")` por `NSFont.systemFont(ofSize: 11, weight: .semibold)`.

### Ícone (novo glifo "anel de 3/4")
- Substituir `PhosphorDrop.swift` por `GaugeRing` (Shape/Path SwiftUI desenhando o anel de 3/4 com indicador).
- Usar em: header do popover, sign-in row, settings header, e como fonte do ícone na barra de menu (`code.svg` → novo SVG do anel) e do `AppIcon`.
- Cor: `t1` (branco). Sem laranja.

### Superfície e janela
- Remover o `ZStack` com `.ultraThinMaterial` e o `SquareWindowFix` (NSViewRepresentable que zera o cornerRadius).
- Fundo do popover: `Color(hex: 0x161719)` sólido. Cantos arredondados nativos do `MenuBarExtra` (16px), borda `outline-2`, sombra do painel.

### Header
- Glifo do anel (`t1`) + "Gauge" (SF Pro 13, semibold, `t1`) + refresh + engrenagem (`t3`).
- Padding 14/11.

### MeterRow (sessão / semanal)
- Label: SF Pro 10, semibold, uppercase, tracking .07em, `t3`.
- Número grande: SF Pro 40, semibold, cor do semáforo conforme %; "%" 17 semibold mesma cor a 70%.
- `SegmentedBar`: 30 segmentos, preenchidos na cor do semáforo, vazios `rgba(255,255,255,.09)`.
- Texto de reset: "Renova em Xh Ym · <quando>" em `t2`/`t4`. (Strings em pt-BR.)

### Cost strip (Hoje / Este mês)
- Mantida. Labels `t3` uppercase; valor SF Pro 16 semibold tabular; tokens `t4`.
- Divisória vertical `outline-2`.
- **Some sozinha** quando não há dados (`isEmpty`), comportamento já existente.

### Status / footer
- Status: bolinha verde (`#5fb07a`) + "Atualizado agora / às HH:MM" em `t3`.
- Footer: "Abrir Claude.ai ↗" e "Sair", SF Pro 12 medium, `t2`, divisória `outline-2`.

### Estados secundários (mesmos tokens)
- **Sign-in row**, **SettingsPanel**, **CodeEntryPanel**, **errorRow**, **loadingRow**: reescrever as strings para pt-BR e aplicar tokens (mono dark, SF Pro, cantos, sem laranja). O vermelho de erro usa `#cc6b61`.

### Barra de menu (`LabelRenderer` em `SiphonApp.swift`)
- Mantém o formato "34% · 82%" com as cores do semáforo dessaturado.
- Ícone tintado para mono (branco no dark, preto no light) — trocar `code.svg` pelo SVG do anel.
- `usageColor()`: atualizar para os hex dessaturados.

---

## Renomeação Siphon → Gauge

| Onde | Ação | Risco |
|---|---|---|
| Strings de UI ("Siphon" → "Gauge") | trocar | nenhum |
| `CFBundleDisplayName` (nome no Finder/menu) | "Gauge" | nenhum |
| Path de credenciais `~/.config/siphon` → `~/.config/gauge` | trocar em `TokenStore` | nenhum (máquina não tem creds ainda; sem migração) |
| Nome do target/scheme/`.xcodeproj` interno | **manter "Siphon"** por ora | renomear no `project.pbxproj` é frágil na mão; invisível ao usuário. Deixar para depois |
| Volume/arquivo do DMG | `Gauge.dmg` | nenhum |

Decisão: o usuário vê "Gauge" em todo lugar; o nome interno do projeto Xcode continua "Siphon" para não arriscar o build. Renomear o projeto fica como melhoria opcional futura.

---

## Build e empacotamento

**Pré-requisito:** instalar o **Xcode completo** (App Store) — a máquina hoje só tem Command Line Tools. O usuário instala; o Claude faz o resto.

Pipeline (já documentado no README do Siphon, validar e adaptar para Gauge):
```
xcodebuild -project Siphon.xcodeproj -scheme Siphon -configuration Release \
  -derivedDataPath build CODE_SIGN_IDENTITY="-" CODE_SIGNING_REQUIRED=NO
# empacotar build/.../Release/<App>.app em Gauge.dmg via hdiutil (com link p/ /Applications)
```
- O `.dmg` sai **sem assinatura/notarização**. Primeiro launch exige "botão direito → Abrir" (Gatekeeper). Aceitável para uso pessoal.
- Distribuir para terceiros exigiria conta Apple Developer (US$99/ano) + notarização — **fora de escopo**.

---

## Limitações conhecidas / riscos

1. **Custo local pode ficar sempre vazio.** Os arquivos `~/.claude/readout-cost-cache.json` e `readout-pricing.json` não existem nesta máquina. A metade "Hoje/Este mês" só aparece se/quando o Claude Code escrever esses arquivos; senão, some sozinha. A metade OAuth (cota de sessão/semanal) funciona normalmente.
2. **Build depende do Xcode.** Se o download de 7GB for inviável, o plano B (SwiftPM + script, só com CLT) é o fallback.
3. **App não assinado.** Aviso do Gatekeeper no primeiro launch.

---

## Verificação

- Compila limpo em Release; `.dmg` monta e o `Gauge.app` arrasta para Applications.
- App roda como item de barra de menu (sem ícone no Dock; `LSUIElement`/MenuBarExtra).
- Popover abre com o visual novo; cota carrega após login OAuth; cores do semáforo corretas nos três limiares.
- Sem referências remanescentes a "Inter", "PhosphorDrop", `.ultraThinMaterial`, `SquareWindowFix` ou "Siphon" visível ao usuário.
