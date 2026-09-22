0. Obiettivo del Programma

L'obiettivo del programma è la realizzazione di un decodificatore universale da codice Morse a caratteri ASCII (A-Z e Spazio). L'aspetto centrale su cui ho focalizzato lo sviluppo è l'ottimizzazione del recupero dei caratteri: ho evitato di scorrere il dizionario elemento per elemento (tante if- else annidate), implementando invece un accumulatore che permette un accesso diretto e immediato alla tabella dei caratteri. Questo garantisce che il tempo impiegato per decodificare e trovare una lettera sia costante, indipendentemente da quale sia il carattere da stampare, questo presuppone però di creare la tabella con un’opportuna logica che le illustro nel file allegato. (In modo simile a quello che accade nel codice visto in classe con la tabella degli indirizzi.)

Per implementare questo progetto ho dovuto immaginare come potrebbe essere un messaggio morse reale dal punto di vista logico, ho adottato le seguenti convenzioni:


1) Il punto lo si rappresenta con 010 cioè un segnale alto corto di durata 1.

2) La linea la si rappresenta con 01110 cioè un segnale alto lungo di durata 3.

3)Il fine carattere lo si rappresenta con 0000 cioè un silenzio prolungato.

Ovviamente queste convenzioni le deve adottare chi scrive il messaggio, l’obiettivo del mio codice è quello di, dato un messaggio con le convenzioni che le ho descritto, ricavarne l’ASCII corrispondente.

In allegato alla presente mail le invio lo script .asm e alcuni miei appunti presi su tablet che illustrano graficamente come ho costruito la tabella e come ho strutturato un esempio di segnale a partire da un messaggio (trasformandolo prima in punti e linee, poi nella sequenza di uno e zeri, e infine nei byte effettivi da caricare in memoria).

Di seguito riporto una descrizione dettagliata del funzionamento del codice, incentrata sull'uso dei registri e sulla loro evoluzione.

1. Utilizzo dei Registri e Architettura dei Dati
Il programma si appoggia a una precisa suddivisione dei registri per gestire le costanti e le variabili di controllo:

$s3 (Accumulatore): È il cuore del programma. Viene inizializzato con un valore di Start Bit pari a 1. Questa scelta è fondamentale per evitare l'ambiguità delle sequenze che iniziano con lo zero (ad esempio, permette di distinguere PUNTO PUNTO LINEA  da PUNTO LINEA, se lo inizializzo con 0 ciò non sarebbe possibile). A ogni simbolo ricevuto, se ho letto un punto metto in coda uno 0, se ho letto una linea metto in coda un uno.

$s1 (Contatore dell'Impulso Alto): Conta il numero di bit consecutivi impostati a 1. Serve a capire la natura del simbolo sul fronte di discesa del segnale: se l'impulso dura 1 solo bit è un Punto; se dura 3 bit è una Linea.

$t9 (Contatore del Silenzio): Conta i bit consecutivi impostati a 0 per determinare le pause spaziatrici.

$s0 (Soglia): Registro costante impostato a 4. Se il contatore del silenzio $t9 raggiunge questo valore, significa che la lettera è conclusa e si deve procedere alla decodifica.

$s2 e $s4 (Registri che interagiscono con il segnale in ingresso): $s2 funge da puntatore al byte corrente del vettore in memoria, mentre $s4 ospita il byte attualmente sotto analisi.

$s5 e $s6 (Scansione Bit-a-Bit): $s5 contiene la maschera logica (inizializzata a 128) che shifta a destra (srl) a ogni ciclo. Quando gli 8 bit del byte corrente sono terminati, la maschera viene rinizializzata a 128, e si andrà a leggere il byte successivo. $s6 memorizza il risultato del mascheramento AND.

$t3 (Base Dizionario): Registro statico di sola lettura che punta all'inizio del vettore dei caratteri ASCII in memoria dati.

2. Comportamento Dinamico ed Evoluzione dei Valori
Il programma analizza i bit letti uno alla volta, modificando i registri secondo questa logica:

Scansione: Il byte in $s4 (byte letto dal messaggio) viene analizzato bit per bit tramite la maschera $s5. Quando la maschera si azzera vuol dire che abbiamo letto tutti i bit del precedente byte quindi l'indice $s2(puntatore alla memoria dove si trova il messaggio) avanza di un byte (addiu $s2, $s2, 1) per caricare il byte successivo della RAM.

Rilevamento del Segnale Alto: Quando si incontra un bit a 1, il contatore del silenzio $t9 viene azzerato e il contatore dell'impulso $s1 si incrementa di un'unità per ogni bit alto letto.

Elaborazione del Simbolo (Transizione da 1 a 0): Non appena il bit torna a 0 dopo una sequenza di 1, l'impulso è terminato e viene valutato il registro $s1(contatore di 1):

Se $s1 == 1 (Punto): $s3 esegue uno shift a sinistra di una posizione, inserendo uno 0 in coda (equivale a fare $s3 = $s3 x 2).

Se $s1 == 3 (Linea): $s3 esegue uno shift a sinistra e poi un'operazione di OR immediato, inserendo un 1 in coda (equivale a fare $s3 = ($s3 x 2) + 1.

Subito dopo, $s1 viene azzerato per prepararsi al simbolo successivo.

Fine Lettera: Se il segnale rimane a 0, $t9 aumenta ad ogni ciclo. Quando raggiunge il valore di timeout in $s0 (4 zeri consecutivi), significa che la lettera è finita.

Decodifica e Stampa: L'offset finale ottenuto in $s3 viene sommato all'indirizzo base della tabella (addu $t4, $t3, $s3). Il programma carica il carattere corrispondente direttamente in $a0 tramite lbu e lo stampa a video con la syscall 11. Al termine, $s3 viene resettato al valore iniziale di Start Bit 1 e $t9 viene azzerato.

3. Gestione della Fine del Messaggio
Nel blocco .data, il vettore segnale_byte termina rigorosamente con un byte nullo (0x00), che funge da terminatore del messaggio.

Quando l'istruzione lbu $s4, 0($s2) preleva questo valore, il controllo salta immediatamente all'etichetta fine_messaggio. A questo punto, il programma esegue un controllo di sicurezza sull'accumulatore $s3: se l'ultimo carattere inserito non è stato ancora stampato (perché il flusso di bit si è interrotto prima di raggiungere i 4 zeri di timeout), il programma forza un ultimo salto alla routine di stampa per svuotare l'accumulatore ed evitare di perdere l'ultima lettera del messaggio. Subito dopo, tramite la syscall 10, il simulatore viene arrestato in modo pulito.
