# Compilatore FOOL

Link potenzialmente utili:

* Backus Normal Form, [BNF](https://it.wikipedia.org/wiki/Backus-Naur_Form) e [EBNF](https://it.wikipedia.org/wiki/EBNF)

INDICE
_________________________

* [Lezione 24 Ottobre 2022](#lezione-24-ottobre-2022)
* [Lezione 07 Novembre 2022](#lezione-07-novembre-2022)
* [Lezione 14 Novembre 2022](#lezione-14-novembre-2022)
* [Lezione 21 Novembre 2022](#lezione-21-novembre-2022)
* [Lezione 28 Novembre 2022](#lezione-28-novembre-2022)

## Lezione 24 Ottobre 2022

-----------------------------------------------------------------------------
REALIZZAZIONE DI UN LEXER CON ANTLR4
-----------------------------------------------------------------------------

I token che dovranno essere individuati da un lexer vengono espressi in ordine decrescente di priorità.

Il token COMMENT risulta essere "problematico" per il nostro lexer. Viene associato 
ai lessemi costituiti da una stringa che inizia con '/\*', contiene una qualsiasi quantità 
(anche nulla) di caratteri qualsiasi, e termina con '\*/'

Soluzione:

```g4
COMMENT : '/*' .*? '*/' 
```

Dove uso la stella di Kleene non greedy `*?` disabilitando il maximal match.
Per la documentazione [vedi qui](https://github.com/antlr/antlr4/blob/master/doc/wildcard.md).

-----------------------------------------------------------------------------
REALIZZAZIONE DI UN PARSER CON ANTLR4 (SimpleExp.g4)
-----------------------------------------------------------------------------
 
ANTLR4, a differenza di versioni precedenti o altri parser top-down, consente 
di utilizzare grammatiche ambigue come 

```
E -> E+E | E*E | (E) | n
```

dichiarando *esplicitamente* priorità e associatività per i vari operatori. 

## PRIORITA 
L' ordine in cui sono scritte le produzioni di una variabile:
la prima produzione ha la priorità più alta.

## ASSOCIATIVITA
per ogni produzione l' associatività di default è a sinistra. Se prima del corpo 
della produzione si specifica `<assoc=right>`, l'associatività è a destra. 

-----------------------------------------------------------------

Affinchè ANTLR4 effettui interamente il parsing del file in input dobbiamo 
aggiungere il token speciale EOF alla grammatica.

-----------------------------------------------------------------------------
VISITA DEL SYNTAX TREE TRAMITE VISITOR PATTERN: CALCOLO RISULTATO ESPRESSIONE
-----------------------------------------------------------------------------

Possiamo calcolare il risultato di una espressione visitando il relativo albero 
sintattico, che ANTLR4 genera esplicitamente come un albero di oggetti 

Ogni nodo interno dell'albero è di classe "<Something>Context" dove "<something>" (l'iniziale è resa maiuscola) è il nome di una variabile della grammatica. 

In ANTLR4, inoltre, è possibile dare un nome a ciascuna produzione di una 
variabile tramite un tag #nome. 
 
La variabile iniziale "prog" ha un'unica produzione, quindi non è necessario dare 
nomi alle sue produzioni per identificarle.

Ogni nodo interno di un albero sintattico è etichettato con il 
nome di una variabile ed ha come figli i simboli del corpo di una 
produzione di quella variabile. Nell'albero di oggetti che ANTLR4 produce tale nodo interno sarà di classe effettiva "ExpProd1Context", sottoclasse di "ExpContext".

## Lezione 07 Novembre 2022

-----------------------------------------------------------------------------
ESPRESSIONI CON MULTIPLI OPERATORI A STESSO LIVELLO DI PRIORITA' (Exp.g4)
-----------------------------------------------------------------------------

L'utilizzo di grammatiche EBNF (Extended Backus-Naur Form), invece di semplici 
CFG, consente di estendere la grammatica aggiungendo operazioni allo stesso livello di priorità

-----------------------------------------------------------------------------
CALCOLO RISULTATO PER ESPRESSIONI CON MULTIPLI OPERATORI A STESSA PRIORITA' 
-----------------------------------------------------------------------------

Nei nodi degli alberi sintattici della grammatica, 
si possono avere due casi: o i suoi figli sono i nodi di una espressione "exp" "*" "exp", 
o di una espressione "exp" "/" "exp".

-----------------------------------------------------------------------------
VISITA SYNTAX TREE TRAMITE VISITOR PATTERN: GENERAZIONE ABSTRACT SYNTAX TREE
-----------------------------------------------------------------------------

Durante la fase di parsing il compilatore genera il syntax tree. Strettamente legata a tale fase vi e' la consecutiva generazione dell'ABSTRACT syntax tree.

1) Usiamo le classi in AST.java per costruire un visitor dei syntax tree delle
espressioni di FOOL.g4 (chiamato ASTGenerationSTVisitor.java, di cui viene dato un file iniziale).
Tale visitor deve generare un Abstract Syntax Tree fatto di oggetti "Node". 
Ad esempio per il programma FOOL
"(4+2)*-5;"  
il visitor deve generare un oggetto fatto come segue:

```java
new ProgNode(
  new TimesNode(
    new PlusNode(
      new IntNode(4), 
      new IntNode(2), 
    ),
    new IntNode(-5), 
  )
)
```
2) Testiamo il visitor tramite Test.java (che lo invoca dopo il parsing).

-----------------------------------------------------------------------------
VISITA ABSTRACT SYNTAX TREE (AST) TRAMITE VISITOR PATTERN: STAMPA DELL'AST 
-----------------------------------------------------------------------------

Facciamo una semplice visita dell'AST generato (composto da oggetti delle classi in AST.java) in modo da visualizzarlo stampando il nome della classe dei suoi nodi (senza il suffisso Node) in modo indentato. 
Per esempio per il programma FOOL considerato prima
"(4+2)*-5;"  
il visitor deve stampare:

```
Prog
  Times
    Plus
      Int: 4
      Int: 2
    Int: -5
```
La classe PrintASTVisitor implementa un metodo "visitNode" per ciascuna classe AST.java,
che riceva un oggetto di tale classe come argomento. 
Invocheremo la visita a partire dalla radice, che e' di classe ProgNode

>Perche' Java da' errore quando proviamo a invocare "visitNode" passando come argomento un nodo figlio?
>
>Mentre, es., in C# esiste un cast "(dynamic)" che consente di determinare il metodo visitNode da invocare a run-time in base al tipo effettivo dell'argomento passato (come dynamic binding ma fatto sull'argomento), cio' non e' possibile in Java: associazione tra invocazione e metodo visitNode invocato fatta a compile-time.

In Java dobbiamo implementare il visitor pattern in modo classico (Come i design pattern comandano!). 

Quando visitavamo i syntax tree utilizzavamo l'implementazione del visitor pattern generata da ANTLR4. Per gli AST abbiamo dovuto implementarlo noi.

-----------------------------------------------------------------------------
VISITA AST CON RITORNO DI UN VALORE (ES. RISULTATO) TRAMITE VISITOR PATTERN  
-----------------------------------------------------------------------------

Questa volta vogliamo realizzare un visitor che torni un valore. 
Come esempio utilizziamo un CalcASTVisitor che calcoli il valore Integer che si ottiene come risultato di un programma FOOL.

Modifichiamo la classe BaseASTVisitor introducendo, tramite i generics di Java, un tipo parametrico S da usare come tipo di ritorno per i metodi visit (tale parametro sara' Integer nel CalcASTVisitor e Void nel PrintASTVisitor). 
Modifichiamo inoltre il metodo accept nell'interfaccia Node e nelle classi dentro AST.java facendo si' che, anch'esso, torni S.

-----------------------------------------------------------------------------
DOTARE VISITE QUALSIASI DI AST DI OPZIONE DI STAMPA PER FARE DEBUG 
-----------------------------------------------------------------------------

In CalcASTVisitor, le stampe sono utili per debug, come posso fare per farle
funzionare? 
Copiare il codice di gestione stampa da PrintASTVisitor non va bene perche' il metodo visit che gestisce l'indentazione funziona con Void (e non Integer) e voglio una soluzione che vada bene per un qualsiasi visitor.

## Lezione 14 Novembre 2022

--------------------------------------------------------------------------------
ESTENSIONE FOOL CON DICHIARAZIONE ED USO DI VARIABILI E FUNZIONI 
--------------------------------------------------------------------------------

ATTENZIONE: gli errori sintattici fanno si' che ANTLR possa fare un match 
parziale delle produzioni: in questo caso i figli successivi all'ultimo figlio 
che fa match sono "null", mentre quelli precedenti sono non-"null".

Alcuni token quindi sono sicuramente non-"null": es. il primo token in 
produzioni che cominciano con un token oppure, ad es., il token PLUS in #plus 
(la ricorsione a sinistra trasformata internamente in destra lo rende primo 
token nel ciclo interno in cui sono matchate le produzioni #plus). 

--------------------------------------------------------------------------------
GENERAZIONE DELL'ENRICHED ABSTRACT SYNTAX TREE (EAST) TRAMITE SYMBOL TABLE
--------------------------------------------------------------------------------

Realizziamo la prima fase della semantic analysis vista a lezione: associare
usi di identificatori (variabili o funzioni) a dichiarazioni tramite symbol 
table, usando le regole di scoping statico
> a use of an identifier x matches the declaration in the most closely enclosing 
scope (such that the declaration precedes the use). 
> A inner scope identifier x declaration hides x declared in an outer scope

In particolare scegliamo la realizzazione della symbol table come lista 
di hashtable.

1) La classe SymbolTableASTVisitorche associa usi a dichiarazioni tramite la symbol table: 
* dando errori in caso di multiple dichiarazioni e identificatori non dichiarati 
* attaccando alla foglia dell'AST che rappresenta l'uso di un identificatore x
  la symbol table entry (oggetto di classe STentry) che contiene le informazioni
  prese dalla dichiarazione di x (per ora consideriamo solo il nesting level)

L'effetto della visita è che l'AST si trasforma in un **Enriched Abstract Syntax Tree** 
(EAST), dove ad alcuni nodi dell'AST sono attaccate *STentry*. 
E' stato possibile stabilire tale collegamento semantico grazie ai nomi degli identificatori, che da ora in poi non verranno più usati.

--------------------------------------------------------------------------------
VISITA ENRICHED AST (EAST) TRAMITE VISITOR PATTERN: STAMPA DELL'EAST 
--------------------------------------------------------------------------------

1) Aggiungiamo un BaseEASTVisitor che estenda il BaseASTVisitor con un metodo di
visita "visitSTentry" con argomento STentry. 

## Lezione 21 Novembre 2022

REALIZZAZIONE DI (ENRICHED) AST VISITOR CHE GENERANO ECCEZIONI
-----------------------------------------------------------------------------

In certi casi, per gestire errori rilevati durante una visita, è necessario 
interrompere la visita lanciando una eccezione (es. perche' è impossibile 
determinare un valore di ritorno consistente per visitNode in caso di errore).

-----------------------------------------------------------------------------
ESTENSIONE INFORMAZIONE CONTENUTA IN STentry: TIPI DEGLI IDENTIFICATORI
-----------------------------------------------------------------------------

Estendiamo le informazioni, prese dalla dichiarazione degli identificatori, 
contenute nelle symbol table entry (classe STentry) introducendo il tipo:
- un tipo BoolTypeNode oppure IntTypeNode per le variabili o per i parametri
- un tipo funzionale ArrowTypeNode (extends TypeNode) per le funzioni

ArrowTypeNode (classe commentata in fondo a AST.java) contiene le informazioni 
corrispondenti alla notazione per i tipi funzionali vista a lezione:
(T1,T2,...,Tn)->T 
Cioè il tipo T1,T2,...,Tn dei parametri (nel campo parlist) ed il tipo T di 
ritorno della funzione (nel campo ret).

-----------------------------------------------------------------------------
TYPE CHECKING TRAMITE VISITA DELL'ENRICHED ABSTRACT SYNTAX TREE
-----------------------------------------------------------------------------

Realizziamo la seconda fase della semantic analysis vista a lezione: il type 
checking, che viene effettuato tramite visita dell'enriched abstract syntax 
tree determinando i tipi delle espressioni (TypeNode) in modo bottom-up.

1) Costruiamo una classe TypeCheckEASTVisitor (di cui viene dato un file 
iniziale) che realizzi il type checking dei programmi FOOL.

La relazione di subtyping è definita tramite il metodo isSubtype di FOOLlib.
Consideriamo i booleani essere sottotipo degli interi con l'interpretazione:
**true vale 1 e false vale 0**.

In caso il visitor rilevi un errore di tipo deve lanciare una eccezione 
TypeException contenente il messaggio di errore ed il numero di linea: ciò 
automaticamente incrementa il contatore "typeErrors" della classe FOOLlib.

3) Per poter rilevare multipli errori di tipo introduciamo la cattura di 
TypeException durante la visita, in caso di type checking di dichiarazioni.
La visita di dichiarazioni non torna un oggetto TypeNode (semplicemente torna 
null) che serva al chiamante: possiamo quindi accettare a questo livello un 
errore di tipo avvenuto dentro la dichiarazione senza propagare l'eccezione.
- introduciamo la cattura e stampa di eccezioni quando si visitano dichiarazioni
- facciamo lo stesso in Test per le eccezioni nella main program expression
 
-----------------------------------------------------------------------------
GESTIONE ERRORI PRECEDENTI A TYPE CHECKING: ST ED (ENRICHED) AST INCOMPLETI
-----------------------------------------------------------------------------

Il compilatore deve completare tutte le fasi del front-end anche in presenza di 
errori collezionando più errori possibili (anche relativi alle fasi precedenti 
al type checking) in modo che il programmatore possa correggerli insieme.

Il problema però è che:
- errori lessicali/sintattici possono portare a creazione da parte di ANTLR4 di 
Syntax Tree incompleti (in cui le variabili figlie di un nodo sono "null")
- tali errori (per un effetto a catena) ed errori semantici rilevati via symbol 
table possono portare alla creazione di EAST che contengono anch'essi figli null
Ciò tipicamente genera null pointer exceptions durante l'esecuzione e impedisce 
che si continui a collezionare errori per le "parti buone" del programma.

1) Gestiamo i Syntax Tree incompleti tornando null quando si effettua una 
qualsiasi visit con argomento null nell'ASTGenerationSTVisitor
(questo risolve il problema ma causa la generazione di AST incompleti)

La gestione degli (Enriched) AST incompleti dipende dallo specifico visitor:
- per alcuni visitor è sufficiente lo stesso approccio usato per i Syntax Tree
(es. SymbolTableASTVisitor e PrintEASTVisitor)
- per altri visitor è necessario gettare un'eccezione unchecked IncomplException
(es. TypeCheckEASTVisitor)

2) Gestiamo i due casi sopra introducendo un parametro booleano "incomplExc" 
aggiuntivo al BaseASTVisitor e al BaseEASTVisitor che indichi se si vuole che 
venga, o meno, gettata una IncomplException in caso di albero incompleto.

- in BaseASTVisitor, quando si effettua una qualsiasi visit con argomento null, 
torniamo null o lanciamo l'eccezione sulla base di tale paramentro
- settiamo appropriatamente tale nuovo parametro in ciascun visitor e in 
TypeCheckEASTVisitor aggiungiamo la cattura di IncomplException quando si 
visitano dichiarazioni e in Test.java

Si noti che la visita dei TypeNode (che ritorna null) è importante, non solo
per stamparli in caso di debug, ma anche per controllare che non siano incompleti 
prima di utilizzarli!

## Lezione 28 Novembre 2022

-----------------------------------------------------------------------------
CODE GENERATION TRAMITE VISITA DELL'(ENRICHED) ABSTRACT SYNTAX TREE
STEP1: NO DICHIARAZIONE (E CHIAMATA) FUNZIONI
-----------------------------------------------------------------------------

Realizziamo l'ultima fase del compilatore: la code generation, che che viene 
effettuato tramite visita dell'(Enriched) Abstract Syntax Tree determinando i 
tipi delle espressioni (TypeNode) in modo bottom-up.

Tecnicamente la code generation non richiederà di proseguire la visita dell'AST  
visitando anche le STentry. Quindi, pur accedendo alle STentry attaccate ai Node, 
sarà sufficiente un CodeGenerationASTVisitor che genera il codice sotto forma di 
una stringa Java.

Sullo stack si ha quindi il solo AR dell'ambiente globale (in cui vengono 
allocate le variabili dichiarate quando inizia l'esecuzione del programma):

2) Modifichiamo la classe SymbolTableASTVisitor aggiungendo il calcolo e 
l'inserimento dell'offset nelle STentry (campo offset) per i VarNode.

-----------------------------------------------------------------------------
CODE GENERATION TRAMITE VISITA DELL'(ENRICHED) ABSTRACT SYNTAX TREE
STEP2: DICHIARAZIONE (E CHIAMATA) DI FUNZIONI E NESTED SCOPES
-----------------------------------------------------------------------------

2) Modifichiamo la classe SymbolTableASTVisitor aggiungendo il calcolo e 
l'inserimento dell'offset nelle STentry (campo offset) per tutte le 
dichiarazioni: variabili, funzioni e parametri.

La code generation di usi di identificatori (usi di variabili IdNode o chiamate 
di funzioni CallNode) richiede di conoscere la differenza di nesting level tra 
l'uso (IdNode/CallNode) e la relativa dichiarazione (campo "nl" della STentry). 
Dobbiamo quindi dotare anche IdNode e CallNode di un campo "nl".

3) Modifichiamo la classe SymbolTableASTVisitor aggiungendo l'inserimento 
del nesting level nel campo "nl" di IdNode e CallNode.

