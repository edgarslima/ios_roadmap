# Lint e Estilo de Código no iOS: do consenso à automação que não atrapalha

Há um ponto, em todo time iOS que cresce, em que a conversa sobre código deixa de ser “gosto pessoal” e vira engenharia de comunicação. Lint e formatação não existem para punir desenvolvedores; existem para tornar o código previsível, legível e fácil de revisar. A regra é simples: se a equipe precisa discutir onde vai a vírgula, a energia não está indo para o produto. Este capítulo mostra como transformar diretrizes de estilo em prática contínua usando **SwiftLint** (linter) e **SwiftFormat** (formatador), amarrando tudo ao **Xcode**, ao **CI** e ao **processo de revisão** sem gerar atrito. O objetivo é que você saia com um fluxo reprodutível que mede, corrige e educa — e que pareça invisível quando está funcionando.

## Estilo é contrato: converta diretrizes em regras executáveis

A referência conceitual de estilo em Swift são as **Swift API Design Guidelines**, que tratam de nomes, clareza sem redundância, uso de verbos, preposições e parâmetros nomeados. A prática mostra que times maduros derivam um **guia interno conciso** dessas diretrizes e o materializam em regras de lint e formatação. Linters verificam **o que** está errado (uso de `force_unwrap`, complexidade ciclomática, nomes confusos), formatadores padronizam **como** o texto é escrito (indentação, quebras de linha, espaços). Em Swift, **SwiftLint** e **SwiftFormat** se complementam: o primeiro aponta inconsistências e problemas de estilo/qualidade; o segundo reescreve o arquivo para que “fique certo” sem esforço manual.

## SwiftLint na prática: instalação, configuração e integração com o Xcode

A forma mais direta de instalar é via **Homebrew** (`brew install swiftlint`) ou adicionando **SwiftLint como dependência de pacote** no Xcode para usar o **Build Tool Plugin**. Com plugin, você acopla o lint diretamente aos *targets* do projeto e evita scripts frágeis; quando o projeto foge do padrão, um **Run Script Build Phase** simples resolve. No script, chame `swiftlint` (ou o executável dentro de `Pods/` se tiver instalado via CocoaPods) e considere a autocorreção segura com `swiftlint --fix && swiftlint` para consertar o que é corrigível antes de reportar violações. Em Xcode 15+, se topar com erros de permissões por sandbox de scripts, desative a sandbox apenas no *target* onde o lint roda; em Xcode 14, a IDE avisa que o script rodará em todo build se o *phase* não tiver “outputs”: você pode desmarcar **Based on dependency analysis** para aceitar esse comportamento. Essas pequenas arestas são conhecidas e documentadas e não devem impedir a adoção.

Crie um arquivo **`.swiftlint.yml`** na raiz do repositório. Comece pequeno, com regras de alto impacto e baixo ruído, e acrescente o resto ao longo do tempo. O segredo é alinhar o que o time quer **ensinar** — e o que o código deve **impedir**.

```yaml
# .swiftlint.yml — ponto de partida pragmático
included:            # pastas a inspecionar
  - Sources
  - App
excluded:            # gere menos ruído
  - Carthage
  - Pods
  - DerivedData

# Torne o resultado previsível para o CI
strict: true         # warnings viram falhas (opcional, time decide)
reporter: "xcode"    # saída amigável no Xcode; 'json' para CI

# Ajuste regras — habilite, desabilite, personalize
disabled_rules:
  - line_length        # habilite depois com limites realistas por contexto
  - cyclomatic_complexity
  - identifier_name    # comece com feedback “coach”, não “polícia”

opt_in_rules:         # regras úteis que não vêm ligadas por padrão
  - empty_count
  - explicit_init
  - force_unwrapping
  - force_cast
  - redundant_nil_coalescing
  - trailing_closure
  - unneeded_parentheses_in_closure_argument

# Parâmetros por regra
force_unwrapping:
  severity: error
type_body_length:
  warning: 300
  error: 500

# Baseline para introduzir lint em projetos grandes sem travar o time
baseline: SwiftLintBaseline.json
```

Adotar **baseline** é o antídoto para “linter- tsunami” em projetos legados: gere um instantâneo das violações atuais e trate o baseline como dívida técnica que será paga aos poucos. O time passa a ser bloqueado apenas por **novas violações**. Quando a arquitetura estabilizar, tire rodinhas: reduza o baseline, promova regras de *warning* para *error* e expanda o conjunto de *rules*.

Para integrar ao Xcode, prefira o **Build Tool Plugin** em *Project → Package Dependencies* anexando o plugin `SwiftLintBuildToolPlugin` ao(s) *target(s)*. Se optar por **Run Script**, adicione um *phase* depois de “Compile Sources” com:

```bash
# Run Script Build Phase — robusto no dia a dia
if [[ "$(uname -m)" == arm64 ]]; then
  export PATH="/opt/homebrew/bin:$PATH"    # Homebrew em Apple Silicon
fi

if command -v swiftlint >/dev/null 2>&1; then
  swiftlint --fix && swiftlint
else
  echo "warning: swiftlint não encontrado — veja README de instalação"
fi
```

Quando precisar **desligar uma regra localmente** para um caso excepcional, use comentários `// swiftlint:disable regra` e `// swiftlint:enable regra` delimitando a menor região possível. E faça *code review* desses “escapes”: se a regra foi desligada com frequência, talvez a regra esteja mal calibrada ou o guia interno precise de nuance.

## Do “guia de estilo” à configuração viva: mapeando diretrizes para regras

As diretrizes de API do Swift pedem nomes claros e concisos, evitando redundâncias óbvias como `get` em *getters* simples, preferindo verbos no imperativo para efeitos colaterais e nomes que leem como frases naturais nos *call sites*. O mapeamento prático em SwiftLint é ligar regras como `identifier_name`, `type_name`, `redundant_string_enum_value`, `redundant_void_return` e `empty_enum_arguments`, além de regras de segurança como `force_cast`, `force_unwrapping` e `implicitly_unwrapped_optional`. A ideia é que o linter eduque: se o código “fala” a língua das diretrizes, ele passa “de primeira”; se não, a ferramenta orienta. Complemente com **SwiftFormat** para padronizar espaços, quebras, vírgulas, `self` implícito/ explícito e dezenas de outras microdecisões que não merecem discussão humana.

Um **`.swiftformat`** mínimo para alinhar forma ao conteúdo pode ser:

```ini
# .swiftformat — formato previsível sem bikeshedding
--indent 4
--maxwidth 120
--commas inline
--self remove
--ifdef no-indent
--swiftversion 5.9
--enable isEmpty
--disable wrapMultilineStatementBraces, redundantFileprivate
```

Rode localmente `swiftformat .` e, se preferir, crie um *hook* de `pre-commit` para formatar antes de cada commit. Ao combinar SwiftLint (análise) com SwiftFormat (formatação), você reduz ruído em PRs e libera o revisor para focar em design e riscos, não em espaços.

## Integração com CI e revisão: o linter como coautor paciente

Lint que só roda na máquina do desenvolvedor não cria confiabilidade de equipe. A bitola é rodar no **CI** e tornar **visível no PR**. Existem duas rotas maduras. A primeira é usar o **`reporter: json`** no SwiftLint e anexar o resultado à *pipeline* (ou falhar o job). A segunda é **integrar com Danger** para comentar violações diretamente na conversa do PR — é didático, dá contexto e aumenta aderência. Em projetos iOS, Danger em Swift já traz integração com SwiftLint pronta para uso e funciona bem em GitHub, GitLab e Bitbucket. Para times com *fastlane*, há uma *action* oficial `swiftlint` que encaixa na *lane* de build/testes.

Um exemplo mínimo de **GitHub Actions** que formata e “linta” poderia ser:

```yaml
name: Lint
on: [pull_request]
jobs:
  lint:
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      - name: Install tools
        run: |
          brew install swiftlint swiftformat
      - name: SwiftFormat (check mode)
        run: swiftformat . --lint --swiftversion 5.9
      - name: SwiftLint
        run: swiftlint --reporter json > swiftlint.json || true
      - name: Upload report
        uses: actions/upload-artifact@v4
        with:
          name: swiftlint-report
          path: swiftlint.json
```

Se quiser **comentários no PR**, configure **Danger Swift** com um `Dangerfile.swift` simples:

```swift
// Dangerfile.swift — comentários automáticos no PR
import Danger

let danger = Danger()
SwiftLint.lint(configFile: ".swiftlint.yml")   // roda SwiftLint e posta comentários
if danger.github.pullRequest.additions ?? 0 > 500 {
  warn("PR muito grande: considere fatiar para facilitar review")
}
```

Rodar o linter no PR muda incentivos. As pessoas recebem feedback enquanto o contexto ainda está fresco, e o histórico fica mais limpo porque problemas triviais não chegam ao `main`. Em paralelo, trate “escapes” com cuidado: comentários `// swiftlint:disable` devem ser raros e explicados no PR; múltiplos casos semelhantes pedem ajuste de regra ou de arquitetura.

## Adoção em projetos grandes: baseline, rollout por módulo e pedagogia de regra

Em bases extensas, **não** ligue tudo no primeiro dia. Crie um **baseline** para estabilizar, habilite 5–10 regras de alto valor, e **plano de rollout** por módulo/camada. À medida que dívidas vão sendo pagas, converta *warnings* em *errors* e remova o baseline. Tenha um documento curto que explique **o porquê** de cada regra relevante (por exemplo, “proibimos `force_unwrapping` fora de inicializadores por segurança; use *guard let* ou `preconditionFailure` quando a falha for programática”). O linter vira professor particular: insiste, mas explica, e não para o time inteiro por ruído.

## Oficina dirigida: colocando tudo para rodar em um app real

Crie um app iOS vazio no Xcode. Instale **SwiftLint** (Homebrew ou Package Dependency com plugin). Adicione o *Run Script* indicado acima e verifique a saída no *Issue Navigator* quando introduzir uma violação proposital, como um `force_try`. Crie o `.swiftlint.yml` com `opt_in_rules` e **gere um baseline** rodando `swiftlint --reporter json > SwiftLintBaseline.json` (ou usando o comando dedicado a baseline da sua versão). Em seguida, instale **SwiftFormat** e rode `swiftformat .` para normalizar a base. Suba um PR com algumas violações e configure **Danger Swift** com `SwiftLint.lint`. Observe os comentários automáticos e ajuste as regras até que o ruído seja aceitável. Por fim, crie um **workflow** no CI como o exemplo acima e valide que o PR falha quando há violações que você marcou como erro.

## Fechamento: menos bikeshedding, mais produto

Quando o estilo deixa de ser negociado a cada arquivo e passa a ser **tratado pelo ferramental**, o time ganha foco e velocidade. SwiftLint transforma diretrizes em feedback automático, SwiftFormat elimina diferenças cosméticas, o Xcode integra isso ao seu ciclo de build, e o CI/Danger leva a disciplina para o PR. O resultado prático é menos discussão sobre estilo e mais conversa sobre modelagem, concorrência, testes e experiência do usuário. Essa é a medida que importa: se o seu linter some do radar, é porque ele está fazendo o trabalho certo.
