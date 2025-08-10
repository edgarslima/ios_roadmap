# Noções de Objective‑C e Interoperabilidade com Swift

Objective‑C é o idioma histórico do ecossistema Apple. Mesmo que sua aplicação seja escrita majoritariamente em Swift, entender os fundamentos da linguagem, o modelo de execução (*runtime*) e como a ponte entre os dois mundos funciona é o que separa o desenvolvedor que “usa frameworks” daquele que domina a plataforma. Este capítulo foca em o que realmente importa para ler código legado com conforto, depurar problemas difíceis e, principalmente, projetar APIs Objective‑C que “soam como Swift” quando importadas, além de expor Swift para código Objective‑C de forma segura e idiomática. A tônica é pragmática: menos nostalgia, mais interoperabilidade que destrava projetos.

## Um retrato fiel do Objective‑C: objetos, mensagens e *runtime* dinâmico

Objective‑C é um superset de C com um sistema de objetos inspirado em Smalltalk. Em vez de chamadas diretas a funções, você **envia mensagens** a objetos, e o *runtime* decide em tempo de execução qual implementação atenderá. Essa natureza dinâmica habilita padrões como *target‑action*, *respondsToSelector:* e a onipresença de *selectors* (`SEL`), e explica por que alguns recursos (como KVO) exigem despacho dinâmico. A referência canônica continua sendo a documentação “The Objective‑C Programming Language”, que cobre a sintaxe de envio de mensagens e a ideia de **ligação dinâmica** (*dynamic binding*). citeturn5search6turn5search1turn5search4

É útil enxergar categorias e extensões como mecanismos de evoluir APIs em código fechado: **categories** adicionam métodos a classes existentes sem subclassificação, enquanto **class extensions** permitem declarar requisitos privados num segundo bloco da mesma classe. Tudo isso é suportado pela mesma infraestrutura do *runtime*, registrada em estruturas e funções públicas do “Objective‑C Runtime”. Conhecer esses alicerces não é opcional quando você precisa depurar comportamento de dispatch, introspecção ou *method swizzling*. citeturn5search2turn5search3

## O que Swift “herda” de Objective‑C na prática

Ao importar frameworks do sistema, Swift traduz objetos Objective‑C para tipos idiomáticos, ajustando nomes, *nullability* e genéricos leves. Uma consequência direta é que você consegue escrever código Swift “natural” consumindo APIs antigas, enquanto mantém compatibilidade binária e semântica do lado Objective‑C. A própria documentação oficial descreve como **APIs C/Objective‑C são importadas**, incluindo como *lightweight generics* e qualificadores de nulidade são mapeados. Essa tradução não é simétrica: algumas construções Swift não têm representação Objective‑C. citeturn0search7

### Nulabilidade e tipos opcionais como contrato estático

O Objective‑C tradicional não carregava informação de nulidade no tipo, mas **anotações de nulabilidade** (`nullable`, `nonnull`, `null_resettable` ou os equivalentes `_Nullable`, `_Nonnull`) foram introduzidas para indicar intenção nos *headers*. Em Swift, isso é usado para decidir se um parâmetro/retorno vira `T` ou `T?`, elevando a segurança de chamadas. Anote as interfaces públicas, ou encapsule trechos legados com *assume‑nonnull* para reduzir ambiguidade em massa. citeturn0search2

### Genéricos *lightweight* em Objective‑C e a leitura em Swift

Com **lightweight generics** — por exemplo, `NSArray<NSString *> *` — o cabeçalho Objective‑C preserva metadados suficientes para o importador Swift tratar a coleção como `[String]` com tipos precisos, livrando você de *casts* verbosos. O recurso foi criado principalmente para melhorar interoperabilidade com Swift, e não para prover verificação profunda do lado Objective‑C. citeturn0search3turn0search24

## Ponte bidirecional: importar Objective‑C em Swift, e Swift em Objective‑C

Interoperabilidade só funciona bem se você dominar as duas direções. Em projetos de app, **Objective‑C → Swift** acontece via *bridging header*; o oposto (**Swift → Objective‑C**) usa um cabeçalho gerado pelo Xcode.

No sentido **Objective‑C → Swift**, você **expõe cabeçalhos** ao Swift por meio de um arquivo de *bridging header* do alvo (por exemplo, `Projeto-Bridging-Header.h`) e o compila junto. O fluxo é simples: crie o *bridging header* (o Xcode oferece automaticamente ao adicionar um `.m` a um projeto Swift) e *import* os `.h` que deseja ver do lado Swift; para *frameworks*, faça isso no *umbrella header*. A documentação “Importing Objective‑C into Swift” detalha esse pipeline. citeturn1search2turn3search22turn1search13

Na volta, **Swift → Objective‑C** usa o arquivo **`<ModuleName>-Swift.h`** que o Xcode **gera automaticamente** para cada *target*. Esse cabeçalho expõe **apenas símbolos Swift compatíveis com Objective‑C**, como classes que herdam de `NSObject`, `@objc` *protocols* e *enums* baseadas em `Int` sem valores associados; tipos puramente Swift (structs, generics “reais”, enums com associados) não aparecem. Para usar do lado `.m/.mm`, importe esse header e anote as declarações Swift com `@objc` (ou `@objcMembers` quando fizer sentido) para tornar a API visível. citeturn1search1

### Como a ponte reescreve assinaturas e erros

Há uma série de melhorias automáticas quando o *importer* traduz APIs: métodos de fábrica viram *initializers* idiomáticos, sufixos redundantes somem e **métodos que usam `NSError **`** são importados como funções **`throws`** em Swift. O inverso também é contemplado: o guia de **Cocoa errors em Swift** explica a ponte `NSError`↔︎`Error` e recomenda capturar **exceções Objective‑C antes de atingirem Swift** (exceções não são parte do modelo de erros de Swift). citeturn1search4

Quando a tradução automática não fica “redonda”, ajuste os *headers* Objective‑C com **`NS_SWIFT_NAME`** para fornecer nomes idiomáticos e aplique **`NS_REFINED_FOR_SWIFT`** quando quiser expor uma versão mais Swift‑friendly de uma API (por exemplo, trocar parâmetros por ponteiros de saída por um retorno de *tuple* em Swift, mantendo a assinatura Objective‑C original para compatibilidade). citeturn1search7turn3search4

## KVO, seletores e despacho dinâmico: onde `@objc` e `dynamic` são obrigatórios

Muita “mágica” dos frameworks Cocoa depende do *runtime* dinâmico. Se você observar propriedades de um objeto Swift via **KVO**, precisa marcar a propriedade com **`@objc dynamic`** para habilitar despacho pelo *runtime* Objective‑C e permitir *swizzling* da *isa* interna. A própria documentação de “Using Key‑Value Observing in Swift” é explícita nesse requisito. Do mesmo modo, interações baseadas em **`Selector`** e `#selector` são expostas pelo módulo Swift para interoperar com APIs dinâmicas. citeturn6search0turn4search1

Para tornar símbolos Swift **visíveis** ao Objective‑C, adote `@objc` pontual ou `@objcMembers` quando a classe inteira precisa ser exportada; lembre que `dynamic` força uso do mecanismo de despacho do *runtime* Objective‑C, o que é necessário em padrões como KVO e *method swizzling*, mas tem custo e deve ser usado com parcimônia. O manual de atributos da linguagem esclarece as implicações e limites (por exemplo, atores `actor` só podem expor membros `@objc` quando marcados como `nonisolated` ou assíncronos). citeturn4search0turn4search8

## Core Foundation e *toll‑free bridging*: quando o ponteiro é o mesmo, mas a propriedade muda de dono

Entre **Foundation** (Objective‑C) e **Core Foundation** (C) existe a chamada **toll‑free bridging**: muitos tipos compartilham a mesma representação e podem ser lançados de um lado para o outro com *cast* apropriado (`CFArrayRef` ↔︎ `NSArray *`, `CFStringRef` ↔︎ `NSString *` etc.). Essa compatibilidade é antiga e documentada; o lado Swift também explica como trabalhar com tipos Core Foundation. citeturn2search2turn2search0turn2search1

Sob ARC, você precisa **dizer ao compilador** quem é o dono quando cruza a ponte: use **`__bridge`** para transferir o ponteiro **sem** transferência de propriedade; **`__bridge_retained`/`CFBridgingRetain`** quando levar um objeto Objective‑C para Core Foundation **assumindo a posse** (chame `CFRelease` depois); e **`__bridge_transfer`/`CFBridgingRelease`** para mover um CF para o lado Objective‑C e **transferir a posse ao ARC**. A própria documentação da Apple apresenta exemplos completos desses três casos. citeturn7search0

## Diferenças fundamentais que impactam design de API

Swift tem **sistema de tipos estático com opcionais e genéricos “de verdade”**, *value types* e *namespaces* por módulo. Objective‑C aposta no **dinamismo do *runtime***, não possui *namespaces* (daí os prefixos `NS`, `UI`, `CF`, e os prefixos de três letras em bibliotecas internas), e introduziu *lightweight generics* e nulabilidade **como metadados** de interface para melhorar a importação por Swift. Na exportação, o conjunto de tipos Swift **representáveis em Objective‑C** é restrito: classes derivadas de `NSObject`, *protocols* `@objc`, *enums* com *raw type* inteiro e algumas *typealiases*. Isso explica por que *structs* e *enums* com valores associados precisam de *facades* quando consumidos em `.m`. citeturn0search3turn4search11

Outro ponto prático: muitos ganchos de Cocoa (KVC/KVO, *target‑action*, `perform(_:)`) dependem de **seletores** e do modelo de **mensagens** de Objective‑C; em Swift você interage com essas APIs através de `Selector` e `#selector`, com checagens adicionais em tempo de compilação. Essa ponte é bem documentada no tópico “Using Objective‑C Runtime Features in Swift”. citeturn4search1

## Oficina guiada: criando um módulo Objective‑C que “fala Swift”

A forma mais eficiente de internalizar interoperabilidade é construir um pequeno módulo Objective‑C e consumi‑lo de Swift, refinando os *headers* até a API ficar idiomática.

Comece por uma classe de cache com uma interface limpa, **anotada** com nulabilidade e genéricos leves. No cabeçalho Objective‑C, declare *nullability* com uma região padrão e seja explícito quando um retorno pode ser nulo. Adote `NS_SWIFT_NAME` para ajustar o nome importado em Swift, evitando verbos redundantes.

```objc
// MyCache.h
#import <Foundation/Foundation.h>
NS_ASSUME_NONNULL_BEGIN

/// Um cache simples com valor genérico "leve" (metadado de interface)
@interface MYCache<KeyType, ObjectType> : NSObject
- (instancetype)initWithCapacity:(NSUInteger)capacity NS_SWIFT_NAME(init(capacity:));
- (nullable ObjectType)objectForKey:(KeyType<NSCopying>)key;
- (void)setObject:(ObjectType)obj forKey:(KeyType<NSCopying>)key;
@end

NS_ASSUME_NONNULL_END
```

Implemente normalmente em `.m`. Ao importar em Swift, você verá *signatures* enxutas, opcionais corretos e nomes idiomáticos graças às anotações.

```swift
// Uso em Swift (após expor MyCache.h no bridging header)
let cache = MYCache<String, NSData>(capacity: 128)
cache.setObject(data, forKey: "avatar")
let avatar: NSData? = cache.objectForKey("avatar")
```

Se você precisar **expor Swift para Objective‑C**, crie uma classe Swift desenhada para o *runtime* Objective‑C: herde de `NSObject`, use `@objc` nos membros que a outra linguagem deve ver, e importe o **`<Module>-Swift.h`** no `.m` que consome. Lembre que *generics* Swift não cruzam a fronteira; construa APIs *erase‑type* ou façam *boxing* quando necessário.

```swift
// Swift: exposto para Objective‑C pelo header <App>-Swift.h
@objcMembers final class TokenStore: NSObject {
    private var storage: [String: String] = [:]
    func set(token: String, for key: String) { storage[key] = token }
    func token(for key: String) -> String? { storage[key] }
}
```

```objc
// Objective‑C: consumindo a classe Swift exposta
#import "MyApp-Swift.h"

void demo(void) {
    TokenStore *store = [TokenStore new];
    [store setWithToken:@"abc" for:@"user"]; // renomeação de parâmetros conforme regras Swift→ObjC
    NSString *token = [store tokenFor:@"user"];
    NSLog(@"%@", token);
}
```

A documentação oficial “Importing Objective‑C into Swift” e “Importing Swift into Objective‑C” resume os passos e pré‑requisitos para cada sentido da ponte, inclusive as restrições de tipos expostos. citeturn3search22turn4search2

## KVO moderno na prática: exigindo `@objc dynamic` e `#keyPath`

Para ilustrar a costura entre os mundos, implemente uma *ViewModel* com estado observado por KVO. Marque propriedades como `@objc dynamic` e utilize `#keyPath` para segurança mínima quando interagir com APIs em string.

```swift
import Foundation

final class Player: NSObject {
    @objc dynamic var progress: Double = 0.0
}

final class Observer {
    private var token: NSKeyValueObservation?
    func attach(_ p: Player) {
        token = p.observe(\.progress, options: [.old, .new]) { _, change in
            print("Progresso: \(Int((change.newValue ?? 0)*100))%")
        }
    }
}
```

A exigência de `@objc dynamic` para observação via KVO é explicitada na documentação de “Using Key‑Value Observing in Swift”; seletores e `#selector` aparecem quando você precisa conectar *target‑action* ou interoperar com APIs baseadas em `SEL`. citeturn6search0turn4search1

## Core Foundation e as três pontes de propriedade

Por fim, exercite o **toll‑free bridging** com transferência de propriedade explícita. No snippet abaixo, criamos um `CFLocaleRef` que segue as **regras de posse Core Foundation** e, em seguida, transferimos a titularidade para ARC com `CFBridgingRelease` ao convertê‑lo para `NSLocale`. O inverso, de `NSLocale` para `CFLocaleRef`, usa `CFBridgingRetain` quando você assume a responsabilidade por liberar com `CFRelease`.

```objc
CFLocaleRef cf = CFLocaleCopyCurrent();                 // você possui "cf"
NSLocale *ns = CFBridgingRelease(cf);                  // ARC passa a possuir "ns"

NSLocale *gb = [[NSLocale alloc] initWithLocaleIdentifier:@"en_GB"];
CFLocaleRef cfGB = CFBridgingRetain(gb);               // você passa a possuir "cfGB"
// ... usa cfGB ...
CFRelease(cfGB);                                       // libera posse explícita
```

Esses três casos — `__bridge`, `__bridge_retained`/`CFBridgingRetain` e `__bridge_transfer`/`CFBridgingRelease` — são documentados pela Apple com exemplos e notas sobre como o compilador entende convenções históricas (por exemplo, *Get/Copy* retorna com posse). citeturn7search0

---

### Checklist mental para interoperabilidade de verdade

Se uma API **Objective‑C será consumida por Swift**, **anote nulabilidade** com `NS_ASSUME_NONNULL_BEGIN/END`, **use genéricos leves** em coleções (`NSArray<...>`, `NSDictionary<..., ...>`) e **refine nomes** com `NS_SWIFT_NAME`/`NS_REFINED_FOR_SWIFT` quando necessário. Se uma API **Swift será consumida por Objective‑C**, **herde de `NSObject`**, **aplique `@objc`/`@objcMembers`** e **confira `<Module>-Swift.h`** no chamador `.m`. Para **KVO/KVC** ou *target‑action*, **marque membros com `@objc dynamic`** e use **`Selector`/`#selector`**. Para **CF↔︎Foundation**, aplique as **três pontes de propriedade** com critério. Para **erros**, prefira a ponte `NSError**`↔︎`throws` e **nunca deixe exceções Objective‑C cruzarem para Swift**. Referências oficiais: importadores de C/Objective‑C, nulabilidade, genéricos leves, KVO em Swift, erros Cocoa e atributos Swift. citeturn0search7turn0search2turn0search3turn6search0turn1search4turn4search0
