# Instruments (Perfilação e Otimização) no iOS: do insight à correção

Instruments é o laboratório de performance da plataforma Apple: uma aplicação que grava a execução do seu app e a projeta como uma trilha temporal cheia de pistas — CPU, memória, energia, eventos de interface, *render loop*, chamadas de sistema. O objetivo deste capítulo é transformar você em alguém capaz de ligar hipóteses a evidências: reproduzir um problema de fluidez, capturar uma *trace*, navegar pelos gráficos e *call stacks*, concluir com precisão onde está a causa raiz, corrigir e validar. Vamos trabalhar em profundidade três frentes decisivas: vazamentos e ciclos de retenção de memória, tempo de CPU e responsividade com Time Profiler e Hangs, e fluidez de UI com Animation Hitches/Core Animation; e costurar esses achados com o **Debug View Hierarchy** do Xcode para inspecionar a árvore de views quando o problema for visual. O tom é prático e avançado: teoria suficiente para entender o que você está medindo, e um método repetível que cabe no seu dia a dia.

## Abrindo a caixa de ferramentas e adotando um método

Você pode abrir o Instruments pelo Xcode (Product → Profile) ou diretamente como app independente. Em ambos os casos, comece escolhendo um **template** que materialize sua pergunta: *Time Profiler* para saber “o que consome CPU e onde”, **Allocations/Leaks/Zombies** para memória, **Animation Hitches** para travadinhas de rolagem e animação, **Hangs** para UI que congela e **Points of Interest** para correlacionar código com eventos por meio de *signposts*. Um trace é um documento vivo: pressione *Record*, execute a tarefa suspeita, pare, e então leia as trilhas de cima para baixo, das visões mais agregadas aos detalhes do *call stack*. Adote uma disciplina simples: formule a hipótese (“o *drop* de FPS ocorre ao abrir a lista”), colete evidência (gravar a *trace* com o template adequado), valide a hipótese com dados (picos, barras vermelhas, amostras que batem com seu código), aplique uma correção mínima e **regrave** para comparar antes/depois.

## Vazamentos de memória e ciclos de retenção sem mistério

O ponto de partida é o **Allocations**: ele registra cada alocação e liberação, mostrando curvas de memória ao longo do tempo. Um crescimento que não volta a um platô após o término de uma ação sugere vazamento. Combine com **Leaks** para detectar blocos alocados que ficaram inacessíveis e com **Zombies** quando você vê *EXC_BAD_ACCESS* e suspeita de mensagens enviadas a objetos já desalocados. Antes de recorrer a ferramentas, vale revisar no código *patterns* que causam vazamento em Swift moderno: capturas fortes em *closures* long-lived (`[weak self]` ausente), ciclos entre objetos que se referenciam mutuamente (ex.: `ViewModel` → `delegate` → `ViewModel`), *timers* e `NotificationCenter` sem *invalidation* ou *removeObserver`, e *bridges* Core Foundation sem transferência de posse correta.

Para sentir na prática, crie um vazamento controlado com um *timer* que retém a si mesmo:

```swift
final class LeakyController {
    var tick: (() -> Void)?
    func start() {
        // Captura forte implícita de self → ciclo com 'tick'
        tick = {
            print(Date())
            self.tick?() // chama de novo por engano
        }
        tick?()
    }
}
```

Grave com **Allocations + Leaks**. Você verá o heap crescer e *Leaks* marcar entradas com backtrace de onde a alocação ocorreu. Corrija substituindo a captura por `[weak self]` e removendo a recursão, e **repita a medição**. Para objetos que “parecem não morrer”, abra o **Memory Graph Debugger** do Xcode durante a execução e navegue pelo grafo de retenções: ele mostra quem mantém quem, onde há referências fortes/ fracas, e aponta ciclos com ícones visuais. Quando o crash for um acesso após desalocação, execute com **Zombies**: a *trace* identifica a classe do “zumbi” e o método acionado, e a partir daí você encontra a origem exata no seu código.

## Tempo de CPU, *main thread* e *call stacks* que contam uma história

**Time Profiler** amostra periodicamente o *call stack* de todas as *threads* e correlaciona consumo de CPU com funções específicas. Para diagnosticar travamentos visíveis, concentre-se na **Main Thread**: se ela passa muito tempo em tarefas pesadas (parsing/IO/decodificação de imagem/compressão), a UI perde quadros. Gravar uma *trace* durante a interação problemática e, em seguida, filtrar pela *Main Thread* revela *hot paths* e *inclusive time* (“esta função é responsável por X% do tempo”). Expanda a árvore até encontrar a sua função de alto custo; se for uma operação inevitavelmente pesada, mova-a para *background* com Swift Concurrency; se for uma iteração redundante, otimize os algoritmos ou a forma de preparar dados. Para perguntas do tipo “quantas vezes esta função roda?” ou “qual a duração de cada execução?”, complemente com **os_signpost** e o instrumento **Points of Interest**, marcando início/fim e medindo latências reais. O resultado é um gráfico sincronizado: picos de CPU e *signposts* nomeados que apontam diretamente para seu código.

```swift
import os.signpost

let log = OSLog(subsystem: "com.acme.app", category: "pricing")

func price(items: [Item]) {
    let id = OSSignpostID(log: log)
    os_signpost(.begin, log: log, name: "Price Pipeline", signpostID: id)
    defer { os_signpost(.end, log: log, name: "Price Pipeline", signpostID: id) }
    // ... trabalho pesado aqui ...
}
```

Com os signposts no lugar, regrave com **Points of Interest + Time Profiler** e correlacione cada fase com o custo observado. Esse hábito cria “telemetria explicável” que acelera regressões futuras.

## Fluidez de interface, *render loop* e o caça-hitches

Em interfaces modernas, o gargalo raramente é “CPU alta” genérica; é o **render loop** perdendo prazos de 16,67 ms (60 Hz) ou 8,33 ms (120 Hz). O template **Animation Hitches** foi desenhado para isso: ele mede *hitch time ratio* e anota eventos em três fases do render (preparação/commit/render). Uma rolagem que “trava e pula” deixa trilhas claras: barras vermelhas em períodos de *hitch* e amostras que, ao serem abertas, mostram operações caras no momento errado — cálculo de *layout* complexo em `cellForRowAt`, conversão de imagens em tempo de scroll, camadas com *offscreen rendering* involuntário ( sombras sem rasterização, máscaras custosas), ou *Auto Layout* re-resolvendo tudo a cada quadro. Use o **Time Profiler** em conjunto para ver *stacks* amostrados durante o hitch e, quando houver suspeita de *rendering*, acione o instrumento de **Core Animation** e verifique métricas como *frames missed* e *rasterization*. A correção costuma ser *pull-based*: pré-computar e cachear o que é determinístico, reduzir profundidade de hierarquias, preferir imagens já decodificadas na rolagem e mover trabalho para fora da janela crítica. Regrave, compare o *hitch ratio* e só dê o *merge* quando a evidência mostrar que o problema sumiu.

## Quando o problema é “visual”: inspecionando a hierarquia de views com precisão cirúrgica

Embora o Instruments seja soberano para tempo e memória, a inspeção da **hierarquia de views** acontece no **Debug View Hierarchy** do Xcode. Pare a execução no momento crítico, capture a hierarquia e use a visualização tridimensional para entender como as subviews se empilham, quais *constraints* regem posições e tamanhos, e onde conteúdos estão sendo **clipped**. O painel de inspeção revela *frames/bounds*, prioridades de Auto Layout e vínculos entre *anchors*. Combine isso com uma *trace* de **Animation Hitches**: primeiro identifique o trecho exato da rolagem em que o hitch acontece e, logo depois, capture a hierarquia para verificar se o custo vem de *layout* excessivo ou de uma pilha de views mais profunda do que parece. Ajuste *constraints*, remova camadas desnecessárias, ative rasterização onde couber — e valide de novo com o Instruments. Esse pingue-pongue entre **View Debugger** e **Instruments** é o atalho mais rápido para apagar *jank* persistente.

## Oficina 1 — Encontrando e consertando um vazamento real

Implemente uma tela que apresenta e dispensa modais rapidamente. Dentro do modal, crie um `Timer` que dispara a cada 0,1 s e atualiza um contador na tela. Esqueça propositalmente de invalidar o `Timer` no `deinit`. Rode com **Allocations + Leaks**, pressione o botão várias vezes e observe a memória em dente-de-serra que só sobe; *Leaks* mostrará entradas vermelhas com *backtrace* apontando para a criação do `Timer`. Corrija adicionando `timer.invalidate()` em `viewWillDisappear` e zere referências fortes a *closures* com `[weak self]` quando necessário. Regrave e confirme que a curva volta ao platô após fechar o modal.

## Oficina 2 — Matando hitches de rolagem numa lista de imagens

Monte uma lista com 500 imagens remotas grandes e, de propósito, converta-as para `UIImage` e redimensione **no `cellForRowAt`**. Grave com **Animation Hitches** enquanto faz *scroll*. Você verá hitches no commit/render. Em seguida, mova a decodificação para um *pipeline* assíncrono com cache, gere *thumbnails* do tamanho exato fora do caminho quente e carregue versões já decodificadas nas células. Regrave: o *hitch ratio* deve cair drasticamente. Se ainda houver *jank*, use **Time Profiler** para conferir filtros/camadas onerosas e ajuste a hierarquia com o **View Debugger**.

## Oficina 3 — Tornando “medível” com *signposts* e validando regressões

Instrumente um fluxo crítico com `os_signpost` marcando as fases “Carregar JSON”, “Parsear”, “Aplicar Layout”. Rode **Points of Interest + Time Profiler** em duas *builds*: antes e depois de uma refatoração. Compare os intervalos pelos *signposts* e o consumo de CPU por função. Promova esse exercício a rotina: perfis curtos e frequentes nos *PRs* impedem que pequenos custos se acumulem até virar um problema visível para o cliente.

## Dicas de leitura e boas práticas que evitam horas perdidas

Crie *schemes* de profilação com **Zombies** e **Malloc Scribble** ativados para sessões de caça a bugs de memória. Prefira **amostragens curtas e focadas**: quanto mais específico o gesto reproduzido durante a gravação, mais claro será o diagnóstico. Em rastros longos, use marcadores do próprio Instruments e *signposts* para voltar exatamente ao ponto de interesse. Em UI, **meça sempre em dispositivo real** — o Simulator ajudará a ilustrar, mas certos gargalos (câmera, GPU, IO) só se revelam no hardware. E, sobretudo, adote a disciplina de **regravar após cada tentativa de correção**: performance sem comparação objetiva vira opinião.

---

### Apêndice: comandos úteis no Console do Instruments/LLDB

Embora o foco seja a interface gráfica do Instruments, às vezes é útil combinar com o console LLDB do Xcode durante a execução do app: `po view.constraints.count` para inspecionar rapidamente o número de *constraints*, `expr UIView.setAnimationsEnabled(false)` para isolar custos de animação, `bt` para confirmar quem chamou um método quente. Ao alternar entre esses dois mundos — rastros temporais no Instruments e inspeção pontual no depurador — você constrói convicção técnica com menos iteração.

### Encerramento

O valor do Instruments está em transformar palpites em fatos reproduzíveis. Quando você observa o crescimento do heap e aponta uma *stack trace* específica; quando mede o *hitch ratio* e associa picos ao seu `cellForRowAt`; quando prova que a *main thread* ficou 40% do tempo em decodificação de JSON durante um gesto de rolagem — você cria um ciclo virtuoso de **hipótese → evidência → correção → verificação**. Domine Allocations/Leaks/Zombies para nunca mais temer memória, torne-se fluente no Time Profiler para “ouvir” o CPU do seu app, e trate Animation Hitches como um check-up de UX. Com esse repertório, otimizar deixa de ser um ato heroico e vira prática cotidiana e objetiva.
