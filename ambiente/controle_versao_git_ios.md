
# Controle de Versão (Git) para iOS em Profundidade

Começar a programar para iPhone e iPad sem um domínio sólido de Git é como pilotar um avião sem instrumentos. O código até decola, mas você não enxerga clima, altitude ou rumo. Git é o painel de controle: registra cada mudança com precisão criptográfica, permite experimentar com segurança, reverter sem pânico e colaborar sem atritos. Neste capítulo, vamos construir uma compreensão profunda e pragmática de Git aplicada ao dia a dia de apps iOS, com uma abordagem que une teoria, prática e a realidade de projetos com Xcode, Swift Package Manager, testes e releases na App Store. O objetivo é que você não apenas “use Git”, mas tome decisões arquiteturais sobre histórico, governança e fluxo de colaboração como alguém que opera em produção e sob pressão.

## Modelo mental: o que realmente acontece quando você comita

Git não armazena “diffs” soltos; ele armazena “fotografias” do estado do seu projeto na forma de objetos. Cada commit aponta para uma árvore que aponta para outras árvores e blobs de conteúdo; isso explica por que o histórico é imutável e por que é tão barato criar branches. Entenda essa ideia com um exercício simples: crie um repositório de teste, faça dois commits mudando um arquivo e investigue o grafo.

```bash
mkdir git-lab && cd git-lab
git init
echo "v1" > versao.txt
git add versao.txt && git commit -m "chore: primeiro snapshot"
echo "v2" > versao.txt
git commit -am "chore: segundo snapshot"
git log --oneline --graph --decorate --all
git cat-file -p HEAD^{tree}
```

Ao explorar os objetos, note que cada commit referencia um “tree” (uma pasta virtual) e, dentro dele, cada arquivo é um “blob”. Essa estrutura de Merkle tree garante integridade: qualquer alteração no conteúdo altera os hashes encadeados, o que torna o histórico confiável. Para iOS, isso significa que você pode ousar: experimente em branches, “refatore até doer” e confie que o passado continua reprodutível.

## Configuração essencial e higiene de repositório em projetos Xcode

Antes de discutir fluxos, padronize a identidade e a ergonomia. Configure nome, e-mail e editor, e ative sugestões de rebase e cores legíveis. Depois, resolva o básico que separa um repo saudável de um repositório barulhento: o `.gitignore` adequado para Xcode e macOS. Ele deve ignorar `DerivedData/`, `xcuserdata/`, artefatos de build, playgounds temporários e demais arquivos de estado de IDE.

```bash
git config --global user.name  "Seu Nome"
git config --global user.email "seu.email@empresa.com"
git config --global core.editor "code --wait"   # ou vim, nano…
git config --global color.ui auto
git config --global pull.rebase false           # manteremos rebase explícito
```

Crie um `.gitignore` específico para Swift/Xcode no repositório e mantenha-o como contrato de time. Inclua seções para SPM, CocoaPods (se aplicável), Carthage e fastlane. Evite ignorar o que influencia o build; ignore apenas o que é gerado ou pessoal. Uma diretriz prática: se a exclusão quebrar um clone “do zero”, esse arquivo não deve estar no `.gitignore`.

```gitignore
# Usuário/IDE
xcuserdata/
*.xccheckout
*.xcscmblueprint

# Build e derivados
DerivedData/
build/

# Playgrounds
timeline.xctimeline
playground.xcworkspace

# Swift Package Manager
.build/

# App bundles e símbolos
*.ipa
*.dSYM.zip
*.dSYM

# fastlane (relatórios e screenshots gerados)
fastlane/report.xml
fastlane/Preview.html
fastlane/screenshots/**/*.png
fastlane/test_output/
```

Para equipes que adotam Swift Package Manager, alinhe uma política sobre o `Package.resolved`: tratá-lo como “lockfile” de versões reprodutíveis costuma ser mais previsível, portanto o fluxo recomendado é **versioná-lo** e atualizá-lo de forma intencional ao subir ou baixar dependências. No Xcode, o `Package.resolved` vive dentro da workspace (`.xcworkspace/xcshareddata/swiftpm/Package.resolved`); garanta que ele acompanhe mudanças de branch para evitar “flapping” de versões. Na prática: após trocar de branch, se o Xcode gritar sobre “Missing package product”, execute “File > Packages > Resolve Package Versions” e confirme que o lockfile foi regenerado, então comite.

## Fluxos de colaboração que funcionam no iOS: curto, claro e audível

Em times pequenos ou squads com forte automação, **Trunk-Based Development** com branches curtíssimos funciona muito bem: feature branch de vida curta, CI agressivo, revisão contínua e merge rápido. Em organizações que operam com ambientes distintos (teste/aceitação/prod), **GitHub Flow** proporciona um caminho enxuto: branch por mudança, pull request, revisão, merge no “main” e deploy contínuo. Quando há múltiplas linhas de produto em produção ou contratos de versão longa, **GitLab Flow** com “release branches” pode dar previsibilidade sem criar “branches eternas”. O denominador comum para apps iOS é evitar branches long-lived que acumulam divergência; quanto menor o delta, menos “merge hell” no `project.pbxproj` e melhor o lead time. Adote metas operacionais: PRs pequenos, revisões em horas (não dias) e integração frequente com testes automatizados rodando no simulador iOS.

## Branching, rebase e merge com histórico limpo e auditável

Branches são ramos descartáveis para isolar ideias, mas o histórico no `main` deve contar uma história legível. Uma estratégia robusta: fazer commits pequenos durante o desenvolvimento, usar `git rebase -i` localmente para “squashar” commits ruidosos, e **squash-merge** no PR para garantir linearidade. Isso facilita reverts, bisect e auditoria. Prefira `git rebase origin/main` antes de abrir o PR para resolver conflitos cedo, mas nunca reescreva histórico já publicado em branches compartilhados. Em repositórios protegidos, rebase e squash acontecem via UI do provedor (GitHub/GitLab), mantendo a trilha de discussão. Na prática do terminal:

```bash
# Atualize seu branch com o main remoto rebaseando
git fetch origin
git rebase origin/main

# Limpeza local antes do push (edite “pick/squash/reword” conforme necessário)
git rebase -i origin/main

# Publica seu branch
git push -u origin minha-feature
```

Se a equipe prefere “merge commit” para preservar a granularidade dos commits do branch, ainda é possível manter clareza: padronize mensagens, exija estado verde no CI e evite merges que tragam commits antigos (“octopus” confuso). Para hotfixes, `git cherry-pick` de commits específicos para branches de release reduz risco e acelera correções críticas.

## Mensagens de commit e versionamento: comunicação é arquitetura

O commit é o menor artefato auditável do seu projeto. Trate a mensagem como uma “mini-justificativa executiva”: linha de assunto concisa (≤ 72 colunas), corpo explicando “por quê” e impactos. Adotar **Conventional Commits** ajuda a padronizar sem burocracia excessiva (`feat:`, `fix:`, `chore:`, `docs:` etc.), além de automatizar changelogs. Para releases de app, use **tags anotadas** (com mensagem e, opcionalmente, assinatura) e **SemVer** quando fizer sentido para SDKs internos. Em apps, as tags do repo mapeiam com a versão do app (CFBundleShortVersionString), o que facilita CI/CD e suporte.

```bash
# Tag anotada e push para origem (ideal para releases)
git tag -a ios-v2.3.0 -m "iOS 2.3.0 - melhorias de performance e correções"
git push origin ios-v2.3.0 --follow-tags
```

## Revisão de código como prática de qualidade (e não ritual)

Revisão boa foca em design, riscos, testes e legibilidade. Padronize expectativas com um template de Pull Request pedindo contexto, riscos, screenshots de UI e plano de testes (simuladores e devices). Combine isso com um arquivo `CODEOWNERS` para que áreas sensíveis (ex.: camada de pagamentos, criptografia, sync) acionem automaticamente revisores responsáveis. Em times iOS, exija que mudanças em `Info.plist`, permissões de `NSCameraUsageDescription` e ajustes de Capability tenham atenção redobrada. A revisão não é para “achar erro de sintaxe”, e sim para promover consistência, reduzir complexidade e antecipar falhas em runtime.

## Integração com Xcode: conflitos previsíveis e como neutralizá-los

O `project.pbxproj` é um arquivo de texto, mas com estrutura propensa a conflitos quando várias pessoas alteram a mesma seção de projeto. Três técnicas reduzem dor: primeiro, **branches curtos e merges frequentes**. Segundo, **usar o mergetool do Xcode** (`opendiff`) quando o conflito for inevitável, pois ele entende melhor a estrutura de projeto. Terceiro, **ensinar o Git a tratar o arquivo com estratégia mais segura**. Em times com conflitos recorrentes, vale experimentar um “merge driver” especializado para `.pbxproj`.

```bash
# Configurar o Xcode como ferramenta de merge
sudo xcode-select -switch /Applications/Xcode.app/Contents/Developer
git config --global merge.tool opendiff
git mergetool  # quando houver conflito, abrirá o assistente do Xcode
```

Você pode ainda testar uma estratégia “union” para arquivos que se beneficiam de união automática, definindo no `.gitattributes`:

```gitattributes
*.pbxproj merge=union
```

Essa abordagem funciona quando as alterações estão em blocos distintos, mas não substitui uma revisão manual. Para equipes grandes, ferramentas como drivers de merge específicos (por exemplo, utilitários dedicados à estrutura do `pbxproj`) podem eliminar classes de conflito comuns. Independentemente da técnica, estabeleça o acordo de que **adicionar arquivos ao target via Xcode** (e não pelo Finder) minimiza mudanças colaterais no `pbxproj`.

## Large files e assets de apps: use Git LFS com disciplina

Repositórios iOS inevitavelmente lidam com imagens, vídeos curtos, modelos Core ML e outros assets pesados. Colocá-los diretamente no Git incha o clone e degrada performance. **Git LFS** resolve rastreando certos padrões (ex.: `*.psd`, `*.mov`, `*.mlmodelc`) e armazenando conteúdo grande em storage otimizado. Integre o LFS cedo no projeto e **publique o `.gitattributes` que define os padrões**. Reserve o versionamento “puro” do Git para código e arquivos de configuração. Uma política madura também define o que *não* deve ir para o repo (ex.: arte-fatos exportados do Xcode Archive, dados de testes volumosos).

```bash
git lfs install
git lfs track "*.psd"
git lfs track "*.mov"
git lfs track "*.mlmodel" "*.mlmodelc"
git add .gitattributes
git commit -m "chore: configura LFS para assets pesados"
```

## Produtividade em monorepos e equipes paralelas: worktree, sparse e partial clone

Se você trabalha em um monorepo iOS com múltiplos módulos, duas técnicas ajudam: **git worktree** para manter duas ou mais branches ativas em diretórios distintos (ótimo para corrigir um hotfix enquanto uma feature segue em curso), e **sparse-checkout/partial clone** para reduzir download e I/O quando o repositório é massivo. No dia a dia, é prático ter um worktree para `release/x.y` e outro para `main` sem ficar trocando de branch e recompilando tudo sempre.

```bash
# Criar um worktree para um hotfix sem largar sua feature atual
git fetch origin
git worktree add -b hotfix/2.3.1 ../app-hotfix origin/release/2.3
# ...trabalhe em ../app-hotfix, rode testes, suba PR…
```

Já o **partial clone** com `--filter=blob:none` baixa árvore e histórico, mas adia o download de blobs até você realmente precisar deles, acelerando clones em repositórios antigos ou cheios de binários acidentais.

```bash
git clone --filter=blob:none git@github.com:suaorg/seu-monorepo.git
# depois, ao trocar de branch/checkout, blobs necessários são baixados sob demanda
```

Em monorepos, combine isso com **sparse-checkout** para materializar apenas pastas do app iOS:

```bash
git sparse-checkout init --cone
git sparse-checkout set apps/ios/MeuApp
```

## Bisect, Blame e cirurgias seguras: achar a causa raiz sem drama

Quando um crash regressivo surge “do nada”, **git bisect** é o bisturi. Ele divide o intervalo entre um commit bom e um ruim, automatiza checagens e aponta o commit ofensivo. Você pode plugar testes automatizados com `xcodebuild test` para tornar o processo irrepreensivelmente objetivo.

```bash
# Supondo que HEAD está “ruim” e abc123 é conhecido como “bom”
git bisect start
git bisect bad
git bisect good abc123
# Em cada passo, rode testes automatizados (ou use --run para automatizar)
xcodebuild -scheme MeuApp -destination "platform=iOS Simulator,name=iPhone 15" test
# Marque como good/bad conforme o resultado até isolar o commit
git bisect reset
```

Para entender “quem e por que” no nível de linha, **git blame** ajuda a contextualizar decisões — mas use com maturidade: o objetivo é insight, não caça às bruxas. Em mudanças arriscadas, **git revert** é o botão de emergência que cria um commit “anti-commit” sem reescrever história, seguro para branches protegidos. Já **git cherry-pick** é seu aliado para levar um fix isolado para uma release prévia sem arrastar mudanças não relacionadas.

## Segurança e conformidade do histórico: commits assinados e branches protegidos

Em apps que manipulam dados sensíveis ou operam em mercados regulados, **assine commits e tags** (GPG/SSH/X.509) e **proteja branches** críticos. Regras como “exigir histórico linear”, “proibir force-push” e “exigir revisões de CODEOWNERS” elevam a integridade do repositório e reduzem risco operacional. Assinar por padrão é trivial: configure uma chave, marque `commit.gpgsign true` e publique a chave pública no provedor Git. Combine com “status checks obrigatórios” para travar merges sem testes passando. Resultado: trilha de auditoria confiável e operações previsíveis.

```bash
# Exemplo com assinatura via SSH (Git 2.34+)
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
git config --global tag.gpgSign true
```

## Integração com práticas de review: templates, checklists e ownership explícito

Padronizar o ritual do PR economiza tempo e eleva a qualidade. Um `PULL_REQUEST_TEMPLATE.md` no repositório pode pedir contexto do problema, decisão de design, plano de testes (incluindo dispositivos e simuladores), impacto em performance e checklist de acessibilidade. O `CODEOWNERS` automatiza convites de revisão para áreas críticas e aciona a regra de “Require review from Code Owners” em branches protegidos. Em iOS, inclua no template campos como “screenshots/Screen Record” e “Checklist de permissões (Camera/Microfone/Photos)” para capturar riscos de runtime que não aparecem em diffs.

## Hooks, automações e qualidade contínua (sem virar burocracia)

Hooks de Git permitem rodar validações na máquina do dev antes do commit/push. Um `pre-commit` pode formatar código, rodar `swiftlint` e checar se o `Package.resolved` mudou sem atualização deliberada. Um `pre-push` pode disparar testes rápidos de unidade no simulador. Tenha parcimônia: valide o essencial localmente e deixe o ciclo pesado para o CI. Ferramentas como “Danger” automatizam comentários em PRs (ex.: “faltou screenshot”, “arquivo grande sem LFS”). Esses guardrails tornam o fluxo previsível sem atritar a equipe.

```bash
# .git/hooks/pre-commit (exemplo minimalista; lembre de dar chmod +x)
#!/usr/bin/env bash
set -euo pipefail
if ! command -v swiftlint >/dev/null 2>&1; then
  echo "swiftlint não encontrado; pulei lint local"
else
  swiftlint --quiet
fi
```

## Operando releases: tags, release branches e rollback confiável

Para cada versão do app, gere **tag anotada** e, se o processo exigir, mantenha um branch de release “congelado” para hotfixes pontuais. Evite backports manuais com “copiar e colar”; prefira cherry-picks claros dos commits específicos. Uma convenção útil é manter sincronia entre o `MARKETING_VERSION` do projeto e o sufixo da tag (ex.: `ios-v3.1.0`). Documente o processo de rollback: como reverter o merge do PR problemático, como “reverter o revert” e como promover o hotfix do branch de release para o `main` (“merge forward” para prevenir regressões).

## Práticas de ouro para times iOS: o que deixa o histórico saudável

Mantenha PRs pequenos e temáticos; evite mudanças de formatação e refactors misturados com features. Revise `project.pbxproj` com atenção redobrada e, quando possível, faça adições de arquivos e alterações de targets de forma atômica para reduzir o delta. Não commit grandes binaries fora do LFS e não versionar artefatos do `Archive`. Trate `Package.resolved` como parte do contrato de build e automatize sua atualização. Proteja `main` e `release/*`, exija checks verdes, revisão de donos e histórico linear. Prefira squash/rebase no merge para clareza e use tags anotadas para releases rastreáveis. Por fim, ensaie incidentes: simule um revert sob pressão e um bisect guiado por testes — ninguém aprende a apagar incêndio durante o incêndio.

## Laboratório dirigido: aplicando tudo em um projeto iOS real

1. **Bootstrap do repo**: crie o app no Xcode com SwiftUI, inicialize o Git, adicione `.gitignore` e um `README` contendo o fluxo adotado. Configure assinatura de commits e branch protection no provedor Git (no mínimo: status checks e revisão obrigatória).  
2. **SPM sob controle**: adicione duas dependências públicas, faça o resolve e comite o `Package.resolved`. Mude de branch e valide que o arquivo permanece consistente. Teste “update” e registre no PR o rational de upgrade.  
3. **Fluxo de PR**: implemente uma pequena feature com uma série de commits “ruidosos”; antes do push, faça `git rebase -i` para “squashar” e “reword” em um único commit semântico. Abra o PR usando o template, adicione screenshots e cubra com testes. Faça **squash merge** no `main`.  
4. **Conflito de `pbxproj` simulado**: crie dois branches adicionando arquivos ao mesmo target; force um conflito e resolva com o mergetool do Xcode. Documente as lições no PR.  
5. **Bisect real**: introduza intencionalmente um bug que quebra um teste. Use `git bisect` com `xcodebuild test` para encontrar o commit culpado, reverta com `git revert` e valide no CI.  
6. **Release com tag**: incremente a versão do app, gere a tag anotada, e faça um cherry-pick de um fix crítico para o branch `release/x.y`. Garanta que o `main` receba o mesmo fix via merge forward.

Ao final desses exercícios, você terá não só o domínio operacional de Git aplicado a iOS, mas um ecossistema de colaboração previsível, auditável e rápido. Esse é o tipo de maturidade operacional que permite que equipes escalem com segurança, reduzam lead time e sustentem a qualidade que usuários de iPhone e iPad esperam.
