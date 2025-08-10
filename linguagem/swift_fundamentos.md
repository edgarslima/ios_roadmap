# Linguagem de Programação Swift e Fundamentos


Swift é uma linguagem moderna desenhada para ser segura por padrão, expressiva no dia a dia e suficientemente performática para cenários de baixa latência. Ao longo deste capítulo, a proposta é construir um entendimento sólido que sirva tanto para escrever o primeiro “Hello, World!” quanto para projetar APIs elegantes, componíveis e seguras sob alta concorrência. Vamos caminhar com progressão didática: iniciar pela sintaxe e pelos tipos que definem a gramática do pensamento em Swift, atravessar opcionais — o mecanismo que eleva segurança a requisito do compilador —, consolidar estruturas, classes e a diferença crucial entre semântica de valor e de referência, dominar protocolos e genéricos como a espinha dorsal de reutilização e polimorfismo, e fechar com closures, gerenciamento de memória com ARC e o modelo moderno de concorrência com async/await e actors. O objetivo é que cada seção reforce a anterior por meio de exemplos reais e exercícios guiados, até que os conceitos “grudem” e virem reflexo.


## Sintaxe essencial, tipos e controle de fluxo

Swift equilibra concisão e legibilidade. Variáveis mutáveis são declaradas com `var` e constantes com `let`, e o compilador pratica inferência agressiva de tipos. O sistema de tipos é estático e forte, o que significa que erros de tipagem são detectados em tempo de compilação e conversões implícitas perigosas não acontecem. Esse desenho incentiva APIs previsíveis e reduz categorias inteiras de bugs.

```swift
let greeting = "Hello"        // String inferida, imutável
var count = 0                 // Int inferido, mutável
count += 1

// Anotações explícitas quando ganham clareza
let pi: Double = 3.14159
```

Coleções padrão — `Array`, `Dictionary`, `Set` — são genéricas e têm semântica de valor com *copy-on-write*, o que dá ergonomia sem custo excessivo na maior parte dos casos. Controle de fluxo usa `if`, `guard` e `switch` com *pattern matching* de primeira classe. `switch` exige exaustividade, o que empurra você a tratar todos os casos e documenta intenções no código.

```swift
let temperatures = [18, 21, 23, 19]
let labels = temperatures.map { t in t > 20 ? "quente" : "amena" }

// guard antecipa falhas e mantém o “caminho feliz” limpo
func read(value: Int?) -> Int {
    guard let v = value else { return 0 }
    return v
}

// switch com padrões potentes
enum Status { case idle, running(progress: Double), failed(Error) }

func report(_ s: Status) -> String {
    switch s {
    case .idle:
        return "Parado"
    case .running(let p) where p < 1.0:
        return "Rodando: \(Int(p*100))%"
    case .running:
        return "Concluído"
    case .failed(let e):
        return "Falhou: \(e.localizedDescription)"
    }
}
```

O operador `defer` permite registrar uma ação a ser executada na saída do escopo atual — útil para liberar recursos determinística e declarativamente.

```swift
func withFileHandle(_ path: String) throws {
    let handle = try FileHandle(forReadingFrom: URL(fileURLWithPath: path))
    defer { try? handle.close() }            // sempre executa
    // ... leitura e processamento
}
```


## Opcionais como contrato de ausência segura

Em Swift, a possibilidade de ausência é um tipo de primeira classe: `Optional<T>` representa “um `T` ou nada”. Esse modelo converte *nulls* em sintaxe explícita (`T?`) e força o pensamento sobre presença/ausência em cada fronteira de dado. Em vez de *null pointer exceptions*, você trabalha com *pattern matching*, *optional binding* e operadores auxiliares, e o compilador impede usos sem verificação.

```swift
var message: String? = nil
message = "Oi"

// Desembrulho seguro
if let text = message {
    print(text.count)
}

// Encadeamento opcional e coalescência nula
let length = message?.count ?? 0

// Map/flatMap sobre opcionais para composição funcional
let upper = message.map { $0.uppercased() }            // Optional<String>
let number = "42".flatMap(Int.init)                    // Optional<Int>
```

Evite o `!` (force unwrap) exceto quando a ausência for logicamente impossível e documentada, como ao sairmos de um *guard* que prova a presença. Para coleções e APIs públicas, prefira modelagens que minimizem opcionais expondo tipos mais precisos — por exemplo, `NonEmptyString` pode encapsular uma `String` válida evitando opcionais a jusante.


## Estruturas, classes e semântica de valor versus referência

A distinção mais estratégica em Swift é entre tipos de **valor** (`struct`, `enum`) e de **referência** (`class`). Valores são copiados na atribuição e passagem de parâmetros (com otimizações *copy-on-write*); referências compartilham identidade e são gerenciadas por ARC. O guideline moderno é privilegiar `struct` por padrão para modelagem de dados imutáveis ou de mutabilidade controlada; `class` fica para casos com identidade intrínseca, herança necesaria, interoperabilidade com Cocoa, ou quando o ciclo de vida precisa de *reference semantics*.

```swift
struct Point { var x: Double; var y: Double }
var p1 = Point(x: 1, y: 2)
var p2 = p1            // cópia
p2.x = 99              // não afeta p1

final class Counter {
    private(set) var value = 0
    func inc() { value += 1 }
}
let c1 = Counter()
let c2 = c1            // mesma instância
c2.inc()
print(c1.value)        // 1
```

Enums com valores associados e *pattern matching* são ferramentas poderosas para estados finitos com dados, empurrando erros para o compilador e documentando transições legíveis.

```swift
enum NetworkState {
    case idle
    case loading
    case success(Data)
    case failure(Error)
}
```


## Protocolos, extensões e polimorfismo orientado a protocolos

Protocolos definem contratos de comportamento; tipos concretos adotam protocolos para ganhar polimorfismo sem herança. Extensões permitem adicionar comportamentos por tipo ou por protocolo, muitas vezes organizando regras por *conformances* específicas. Em Swift moderno, “polimorfismo por protocolo” é o caminho idiomático para desacoplar módulos.

```swift
protocol Cache {
    associatedtype Key: Hashable
    associatedtype Value
    mutating func insert(_ value: Value, for key: Key)
    func value(for key: Key) -> Value?
}

struct LRUCache<K: Hashable, V>: Cache {
    typealias Key = K; typealias Value = V
    private var storage: [K: V] = [:]
    mutating func insert(_ value: V, for key: K) { storage[key] = value }
    func value(for key: K) -> V? { storage[key] }
}

// Existenciais e o modificador 'any' ao expor valores de protocolo em variáveis
func printAll(_ items: [any CustomStringConvertible]) {
    items.forEach { print($0.description) }
}
```

### Genéricos, *associated types* e tipos opacos

Genéricos capturam a forma do algoritmo enquanto adiam a escolha concreta do tipo. Com *constraints* (por exemplo, `T: Sequence`) você escreve funções e estruturas reutilizáveis, e com *where clauses* ajusta comportamentos a subconjuntos de tipos.

```swift
func firstMatch<S: Sequence>(in source: S, where predicate: (S.Element) -> Bool) -> S.Element? {
    for e in source where predicate(e) { return e }
    return nil
}
```

Tipos opacos com `some` expõem “o que algo faz” sem revelar “o que algo é”, permitindo ocultar detalhes de implementação preservando desempenho e checagem estática.

```swift
protocol Shape { func area() -> Double }

struct Circle: Shape { let r: Double; func area() -> Double { .pi * r * r } }
struct Square: Shape { let a: Double; func area() -> Double { a * a } }

func makeShape(_ large: Bool) -> some Shape {
    large ? Circle(r: 10) : Square(a: 3)
}
```


## Funções e closures: capturas, escaping e APIs expressivas

Funções são tipos de primeira classe, e closures são blocos de código auto-contidos que podem capturar variáveis de seu contexto. A sintaxe evolui de formas verbosas a expressões curtas conforme o compilador infere tipos. Capturas são fortes por padrão; em contextos assíncronos ou que possam formar ciclos de referência, declare listas de captura com `weak` ou `unowned` para evitar *leaks*.

```swift
// Sintaxe básica
let add: (Int, Int) -> Int = { (a, b) in a + b }
let squares = [1,2,3,4].map { $0 * $0 }

// Escaping: a closure armazenada para uso futuro
final class Emitter {
    private var handlers: [(String) -> Void] = []
    func onMessage(_ h: @escaping (String) -> Void) { handlers.append(h) }
    func emit(_ text: String) { handlers.forEach { $0(text) } }
}

// Captura fraca para evitar ciclos (self <-> closure)
final class Controller {
    var log: [String] = []
    func start() {
        Emitter().onMessage { [weak self] msg in
            self?.log.append(msg)
        }
    }
}
```

Closures também habilitam padrões como `@autoclosure` para adiar avaliação de expressões e *result builders* (como em SwiftUI) para construir DSLs declarativas. Use com parcimônia e clareza, priorizando APIs que “leem” como linguagem natural.


## Gerenciamento de memória com ARC, ciclos de referência e *copy-on-write*

Swift usa **Automatic Reference Counting (ARC)** para tipos de referência. Cada instância tem um contador de referências fortes; quando ele chega a zero, a memória é desalocada. O compilador injeta incrementos/decrementos automaticamente, mas cabe ao autor da API evitar ciclos de retenção, que ocorrem quando duas instâncias mantêm referências fortes entre si. As ferramentas são `weak` (referência opcional, não retém, zera ao desalocar) e `unowned` (não opcional, não retém, pressupõe vida útil do outro objeto).

```swift
final class User { let name: String; init(_ n: String) { name = n } }
final class Room {
    let id: Int
    weak var owner: User?           // evita ciclo com User
    init(id: Int) { self.id = id }
}
```

Tipos de valor da biblioteca padrão (`String`, `Array`, `Dictionary`, `Set`) aplicam *copy-on-write*: múltiplas cópias compartilham o mesmo buffer até que alguém escreva; só então ocorre uma cópia real. Isso combina semântica de valor intuitiva com eficiência concreta em memória e CPU na média dos casos.


## Inicialização, erros e resultados previsíveis

Inicializadores (`init`) constroem invariantes; você pode ter *memberwise init* em `structs`, inicializadores falháveis com `init?` e *convenience inits* em classes para criar fábricas. O tratamento de erros usa `throws`/`try`/`catch`, empurrando exceções para o tipo ao invés de *magic values*. Para fluxos funcionais ou integração com APIs assíncronas, `Result<Success, Failure>` é útil e explícito.

```swift
enum ParseError: Error { case empty, invalid }

func parseInt(_ s: String) throws -> Int {
    guard !s.isEmpty else { throw ParseError.empty }
    guard let n = Int(s) else { throw ParseError.invalid }
    return n
}

do {
    let n = try parseInt("42")
    print(n)
} catch {
    print("Erro: \(error)")
}
```


## Concorrência moderna: async/await, tasks e actors

Swift adota **concorrência estruturada** com `async`/`await`, `Task` e `TaskGroup`, oferecendo legibilidade de código síncrono com performance assíncrona. O compilador entende pontos de suspensão (`await`) e propaga requisitos de assincronicidade pela assinatura das funções. Para isolar estado sob concorrência, use **actors**: apenas código executando no actor acessa seu estado interno, o que elimina *data races* por construção. A anotação `@MainActor` garante que interações com UI ocorram no executor principal, simplificando integrações com frameworks gráficos.

```swift
actor CounterActor {
    private var value = 0
    func inc() { value += 1 }
    func current() -> Int { value }
}

@MainActor
final class ViewModel: ObservableObject {
    private let counter = CounterActor()
    @Published var text = "…"

    func load() {
        Task {
            await counter.inc()
            text = "Valor: \(await counter.current())"
        }
    }
}

// Paralelismo com TaskGroup
func fetchAll(_ urls: [URL]) async throws -> [Data] {
    try await withThrowingTaskGroup(of: Data.self) { group in
        for u in urls { group.addTask { try Data(contentsOf: u) } }
        var result: [Data] = []
        for try await d in group { result.append(d) }
        return result
    }
}
```

No modo de linguagem Swift 6, a checagem **strict concurrency** promove diagnósticos de possíveis *data races* a erros de compilação e exige modelagem explícita de fronteiras de concorrência com `Sendable`, o que fortalece garantias de segurança em produção. Quando interoperar com GCD/`DispatchQueue`, mantenha o princípio: limite a criação de filas, prefira `Task {}` em código moderno e isole mutações de estado dentro de actors ou do `@MainActor`.


## Exercícios guiados: do rascunho ao músculo de memória

Como prática, implemente uma pequena engine de regras que avalia descontos. Comece com um modelo de valor (`struct`) imutável para `CartItem` e `Order`, modele regras como protocolos com *associated types*, componha com genéricos e aplique concorrência para precificar lotes em paralelo. O foco é exercitar protocolos, genéricos, closures e isolamento de estado.

```swift
import Foundation

struct CartItem: Hashable {
    let sku: String
    let price: Decimal
    let quantity: Int
}

struct Order {
    let id: UUID
    let items: [CartItem]
    var subtotal: Decimal { items.reduce(0) { $0 + $1.price * Decimal($1.quantity) } }
}

protocol DiscountRule {
    associatedtype Input
    associatedtype Output
    func apply(_ input: Input) -> Output
}

// Uma regra concreta: “leve 3 pague 2” por SKU
struct Buy3Pay2: DiscountRule {
    func apply(_ items: [CartItem]) -> Decimal {
        items.reduce(0) { acc, item in
            let free = item.quantity / 3
            let offs = (item.price) * Decimal(free)
            return acc + offs
        }
    }
}

// Composição funcional de regras com genéricos
struct Pipeline<A, B>: DiscountRule {
    let f: (A) -> B
    func apply(_ input: A) -> B { f(input) }

    func then<C>(_ g: @escaping (B) -> C) -> Pipeline<A, C> {
        Pipeline<A, C> { a in g(self.f(a)) }
    }
}

func currency(_ d: Decimal) -> String {
    let f = NumberFormatter()
    f.numberStyle = .currency
    f.locale = Locale(identifier: "pt_BR")
    return f.string(from: d as NSDecimalNumber) ?? "\(d)"
}

@MainActor
final class PricingEngine: ObservableObject {
    private let group = DispatchGroup()   // poderia ser TaskGroup em app real
    @Published var last: String = ""

    func price(_ order: Order) {
        let rule = Pipeline<[CartItem], Decimal> { Buy3Pay2().apply($0) }
            .then { discount in max(discount, 0) }

        let discount = rule.apply(order.items)
        let total = order.subtotal - discount
        self.last = "Subtotal: \(currency(order.subtotal))  Desconto: \(currency(discount))  Total: \(currency(total))"
    }
}

// Exercício: reimplemente a engine usando actors e TaskGroup, marcando tipos que cruzam threads como Sendable.
```


## Encerrando o arco: como pensar em Swift no dia a dia

O ganho real de produtividade em Swift vem de internalizar três ideias: trate ausência como tipo (opcionais e *pattern matching*), modele dados com semântica de valor por padrão (structs/enums) e exponha comportamentos por protocolos genéricos. Feche as arestas com closures expressivas, ARC bem compreendido e concorrência estruturada com actors e `async/await`. Ao seguir as diretrizes oficiais de design de APIs e deixar o compilador trabalhar a seu favor, seu código passa a comunicar intenções tanto para humanos quanto para as ferramentas, reduzindo ambiguidade e aumentando robustez. Daqui em diante, cada framework da plataforma torna-se apenas um vocabulário específico que você fala com fluência porque domina a gramática da linguagem.
