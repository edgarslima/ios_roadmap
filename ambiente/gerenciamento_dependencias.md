# Gerenciamento de Dependências no iOS: Swift Package Manager e CocoaPods na prática

A gestão de dependências é o elo entre a arquitetura que você planeja e o código que você realmente entrega. Sem disciplina, bibliotecas externas viram uma fonte inesgotável de acoplamento invisível; com método, elas aceleram a entrega sem comprometer previsibilidade. Este capítulo constrói um entendimento sólido e pragmático de dois mecanismos dominantes no ecossistema Apple: **Swift Package Manager (SPM)**, integrado ao Xcode e à toolchain do Swift, e **CocoaPods**, veterano do mercado com enorme cobertura de bibliotecas. O foco é desenvolver fluência operacional: você vai entender como cada ferramenta resolve versões, injeta produtos nos seus *targets*, grava arquivos de *lock*, lida com binários, recursos, CI e integração com o Xcode. O objetivo é sair do “funciona na minha máquina” e chegar ao “build determinístico em qualquer máquina do time”, com regras claras de atualização e rollback.

## Swift Package Manager com profundidade: do manifesto ao *build* reprodutível

O SPM é parte da linguagem e conversa nativamente com o Xcode: você adiciona um pacote por **File → Add Package Dependency…**, informa a URL do repositório Git, define a **exigência de versão** e o Xcode injeta os produtos selecionados como dependências do seu *target*. Por baixo, o SPM interpreta o **`Package.swift`** do pacote, resolve versões e gera a malha de dependências. O Xcode cria/atualiza o arquivo **`Package.resolved`** próximo ao seu projeto para registrar exatamente **quais commits** foram escolhidos, de forma que qualquer máquina possa reproduzir o mesmo *resolution graph* sem adivinhar versões. Em pipelines de CI, o mesmo grafo é respeitado, e você controla atualização de dependências rodando o *resolve/update* localmente antes de *commitar*. 

O manifesto é o contrato. Um pacote moderno declara nome, plataformas, produtos, *targets* e dependências. Repare que a especificação de versão usa SemVer e que você pode variar a exigência conforme o risco que aceita assumir — de um intervalo aberto por *major* até um *pin* exato.

```swift
// Package.swift (trecho essencial)
import PackageDescription

let package = Package(
    name: "PricingCore",
    platforms: [.iOS(.v15)],
    products: [
        .library(name: "PricingCore", targets: ["PricingCore"])
    ],
    dependencies: [
        .package(url: "https://github.com/Alamofire/Alamofire.git",
                 .upToNextMajor(from: "5.8.0")),  // intervalo [5.8.0 ..< 6.0.0)
        .package(url: "https://github.com/apple/swift-collections.git",
                 .exact("1.0.4")),                // *pin* exato
        .package(url: "https://github.com/owner/feature-flags.git",
                 .branch("main"))                 // branch (útil em POCs/monorepos)
    ],
    targets: [
        .target(
            name: "PricingCore",
            dependencies: [
                .product(name: "Alamofire", package: "Alamofire"),
                .product(name: "Collections", package: "swift-collections")
            ],
            resources: [.process("Resources")]    // assets via SPM (Swift 5.3+)
        ),
        .testTarget(name: "PricingCoreTests", dependencies: ["PricingCore"])
    ]
)
```

Para **recursos** (imagens, JSON, *storyboards*, bundles), o SPM gera um *bundle* e uma API de acesso a partir de `Bundle.module`, o que elimina gambiarras históricas. Para **binários**, o SPM aceita **`binaryTarget`** apontando para um `.xcframework` local ou remoto, de forma assinada e reproduzível; é a estratégia recomendada para distribuir SDKs fechados que não podem expor código-fonte. Em cenários de automação, **plugins** do SPM permitem acoplar ferramentas de *build* (geração de código, lint, DocC) ao ciclo, mantendo o projeto enxuto e sem *scripts* frágeis. Quando o assunto é **integração com Xcode**, você gerencia pacotes no painel **Project → Package Dependencies**, decide a regra de atualização, marca *exact pins* quando necessário e, em times, **commita o `Package.resolved`** para consolidar o grafo. Quando der ruim — e vai acontecer —, zere caches de pacote, re-resolva e trate conflitos como você trataria um *merge* de código.

### Exemplo de binário via Swift Package

```swift
// Trecho de manifesto com alvo binário (XCFramework entregue fora do repositório)
.target(
    name: "Payments",
    dependencies: [
        .target(name: "PaymentsSDK")  // wrapper para o binário
    ]
),
.binaryTarget(
    name: "PaymentsSDK",
    url: "https://cdn.example.com/PaymentsSDK.xcframework.zip",
    checksum: "b9f1a7..."
)
```

No consumidor, basta `import Payments` no código e adicionar o produto ao *target* no Xcode. Para auditoria, valide `checksum`, mantenha o artefato versionado por SemVer do lado do provedor e atualize o `Package.resolved` no *PR* de bump.

### Boas práticas operacionais com SPM

Em bibliotecas internas, prefira **pacotes locais** (`.package(path: "../Common")`) durante a extração e, após estabilizar, publique em um repositório Git para pin de versão. Em apps, use **`.upToNextMajor`** para dependências maduras e **`.exact`** para SDKs críticos; evite branches em produção. Trate **`Package.resolved`** como parte do contrato de build: *commite*, revise *diffs* e nunca edite manualmente. Em **CI**, use `xcodebuild` ou Xcode Cloud com *automatic resolution* consistente, e agende revisões de segurança de tempos em tempos para atualizar mínimos de plataforma e versões de pacotes.

## CocoaPods com rigor: o que acontece quando você roda `pod install`

CocoaPods parte de um arquivo declarativo, o **`Podfile`**, que descreve dependências por *target* do seu projeto. Ao rodar `pod install`, a ferramenta resolve versões, baixa código-fonte, **gera um projeto `Pods.xcodeproj`** e cria um **`.xcworkspace`** que agrega o seu projeto e os Pods. A partir daí, você **abre sempre o workspace**; é nele que os *build phases* e *search paths* estão corretos para linkar os pods. Além de instalar, o CocoaPods escreve o **`Podfile.lock`**, um instantâneo das versões resolvidas; enquanto você usar `pod install`, essas versões permanecem fixas. Para atualizar, você roda `pod update` (global ou para um pod específico), revisa o *diff* do `Podfile.lock` e valida o build.

Um `Podfile` mínimo é direto; um avançado pode declarar plataformas, *post-install hooks*, estilos de *linkage* e cabeçalhos modulares. O atributo **`use_frameworks!`** hoje aceita **`:linkage => :static | :dynamic`** e deixou de ser obrigatório para projetos Swift desde a versão 1.5 do CocoaPods, que passou a suportar **bibliotecas Swift estáticas**. Para interop complexa C/Obj‑C, *modular headers* ajudam a expor cabeçalhos consumíveis por Swift. Em equipes, erros como “**The sandbox is not in sync with the Podfile.lock**” denunciam que alguém abriu o `.xcodeproj` ou alterou pods sem atualizar o `Podfile.lock`; a solução é reexecutar `pod install` e normalizar o workspace.

```ruby
# Podfile (exemplo avançado)
platform :ios, '15.0'

target 'App' do
  # Sem 'use_frameworks!' por padrão (instalação como estática quando possível)
  pod 'Alamofire', '~> 5.8'
  pod 'SDWebImage', '~> 5.19'

  # Exemplo de escolha explícita de linkage para um pod específico
  use_frameworks! :linkage => :static

  # Caso precise de cabeçalhos modulares para um Pod Objective-C
  pod 'SomeLegacyObjCLib', :modular_headers => true

  target 'AppTests' do
    inherit! :search_paths
    pod 'Quick', '~> 7.0'
    pod 'Nimble', '~> 13.0'
  end
end
```

O ciclo operacional é objetivo: **`pod install`** respeita o `Podfile.lock` existente e apenas instala versões já fixadas; **`pod update`** re-resolve versões de acordo com as restrições do `Podfile` e atualiza o *lockfile*. Em repositórios, **sempre versione o `Podfile.lock`** e o diretório `Pods/` **não** precisa ser comitado (a recomendação padrão é ignorá-lo). Para ambientes novos ou máquinas de CI, bastam `pod install` e a versão da RubyGem alinhada; como sanity check, valide a versão do CocoaPods no time e em runners para evitar *drift* de comportamento do resolvedor.

## Comparando mecanismos sob o ângulo de engenharia

Quando o requisito é integração coesa com o Xcode, versionamento SemVer, **recursos e binários via `.xcframework`**, além de automação por **plugins**, o SPM oferece a rota mais natural e menos *boilerplate*. Ele simplifica a vida para pacotes internos, facilita particionar apps em módulos e reduz atrito em CI por eliminar scripts externos. CocoaPods, por sua vez, ainda brilha em **catálogos amplos de pods existentes** e em cenários com **árvores antigas em Objective‑C/C/C++** que exigem ajustes finos de *search paths* e *header maps*; quando a biblioteca só existe como Pod, a decisão é pragmática. Em ambos os casos, a regra é a mesma: trate o arquivo de *lock* como contrato de build, evite *update* amplo sem revisão e gere métricas de tempo de *link* e tempo de *launch* ao trocar linkage (estático vs dinâmico), porque essas escolhas têm impacto real em tamanho do app e tempo de inicialização.

## Oficina guiada com SPM: do zero ao *pin* reprodutível

Crie um novo app iOS no Xcode e, em **File → Add Package Dependency…**, adicione `https://github.com/apple/swift-collections`. Selecione **Up to Next Major** a partir de `1.0.0` e inclua o produto **Collections** no *target* principal. Confirme a criação do **`Package.resolved`** no seu repositório e rode o app. Em seguida, altere o manifesto do seu pacote interno para adicionar **recursos**: crie a pasta `Sources/PricingCore/Resources` e declare `resources: [.process("Resources")]`; acesse com `Bundle.module.url(forResource:withExtension:)` para carregar JSON de teste. Para fechar, simule um **SDK binário**: crie um pacote “wrapper” com `.binaryTarget` apontando para um `.xcframework` de exemplo, faça o *pin* por checksum e injete o produto no app. Por fim, rode em uma pipeline de CI com `xcodebuild` para garantir que o build é reprodutível fora da sua máquina.

## Oficina guiada com CocoaPods: workspace limpo e atualização controlada

Num projeto existente, instale o CocoaPods (`gem install cocoapods`), rode `pod init` e edite o `Podfile` adicionando `pod 'Alamofire', '~> 5.8'`. Execute `pod install` e **abra o `.xcworkspace`** recém-criado. Faça um *commit* incluindo `Podfile` e `Podfile.lock`. Em seguida, force um cenário de atualização controlada: rode `pod update Alamofire` para atualizar **apenas** esse pod; revise o *diff* do `Podfile.lock`, rode o app e monitore tempo de *cold launch*; teste também trocar o *linkage* para `use_frameworks! :linkage => :static` e mensure impacto de tamanho do app e tempo de carregamento. Finalize provocando um erro comum: delete o `Podfile.lock` e tente *buildar* — ao ver “sandbox not in sync”, rode `pod install` e normalize o estado.

## Fechamento: trate dependências como parte da arquitetura

Escolher entre SPM e CocoaPods não é questão de torcida, e sim de **governança de dependências**. Com SPM, você ganha integração nativa, binários via `.xcframework`, recursos empacotados e plugins que reduzem scripts frágeis; com CocoaPods, você herda a maturidade de um ecossistema imenso e mecanismos sólidos de *lock* e resolução por *target*. Na prática, equipes de alto desempenho convergem para SPM como padrão e mantêm CocoaPods para casos específicos do legado. Independentemente da ferramenta, o que sustenta a qualidade é o processo: **versionar arquivos de *lock***, **limitar superfícies públicas importadas**, **documentar regras de atualização**, **medir efeitos de linkage** e **automatizar o build em CI**. Se você fizer isso com disciplina, dependências deixam de ser um risco silencioso e viram alavanca de velocidade sem surpresas.
