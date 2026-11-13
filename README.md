# Previsione pre-impatto della caduta con un sensore inerziale indossabile

Modello di previsione della caduta pensato per girare **dentro** un sensore inerziale
MEMS, nel Machine Learning Core (MLC) dell'STMicroelectronics LSM6DSV16X, e destinato a
un gilet protettivo che gonfia un airbag prima dell'impatto.

Il modello riconosce la caduta mentre è in corso, nella finestra di tempo fra la perdita
di equilibrio e il contatto con il suolo, e deve lasciare abbastanza tempo al gonfiaggio
dell'airbag, che richiede circa 200 ms.

Questo repository raccoglie il codice e i risultati della tesi di laurea triennale in
Ingegneria dell'Innovazione Digitale, Università del Salento.

## Risultato principale

Albero decisionale di profondità 6 (60 foglie) su quattro caratteristiche ricorsive,
validato con lo schema *leave-one-subject-out* su tutti e 32 i soggetti del dataset
KFall: ogni soggetto viene valutato da un modello che durante l'addestramento non lo ha
mai visto.

| | Albero decisionale | Algoritmo a soglie |
|---|---|---|
| Cadute riconosciute | 95,3 % | 90,2 % |
| Riconosciute con almeno 200 ms di anticipo | 82,2 % | 72,1 % |
| Attivazioni premature (prima dell'inizio della caduta) | 0,9 % | 5,6 % |
| Falsi allarmi sulle attività quotidiane | 27,4 % | 46,2 % |
| Anticipo mediano | 320 ms | 320 ms |

Punto di funzionamento: soglia di probabilità 0,7, persistenza 1 campione.
A parità di falsi allarmi tollerati l'albero resta sopra l'algoritmo a soglie su tutta
la curva di compromesso, con un vantaggio fra i 18 e i 34 punti percentuali sulle
cadute utili.

Il modello è stato poi **caricato ed eseguito dentro il sensore**. Quella verifica ha
mostrato due cose: che il trasferimento funziona, e che fra il modello validato e il
modello eseguito c'è uno scarto importante. I due paragrafi "Dal modello al sensore" e
"Cosa non torna fra calcolatore e dispositivo" lo descrivono.

## Le quattro caratteristiche

Il MLC non ha memoria per conservare una finestra di segnale, quindi le caratteristiche
sono calcolate in forma **ricorsiva**: ogni nuovo campione aggiorna il valore precedente
senza che il passato venga conservato.

    y[i] = ALFA * x[i] + (1 - ALFA) * y[i-1]      ALFA = 0,10

Le quattro grandezze usate dall'albero sono la media ricorsiva e la varianza ricorsiva
del modulo dell'accelerazione e della velocità angolare. Restano piatte durante le
attività quotidiane e si muovono con decisione appena la caduta comincia.

In addestramento queste caratteristiche sono aggiornate **a ogni campione**. Dentro il
sensore, come spiegato più sotto, l'aggiornamento avviene una volta per finestra: è la
differenza all'origine dello scarto misurato.

## Struttura del repository

    modello_caduta.ipynb              l'intero lavoro, dal dataset alle misure sul sensore
    caduta_esempio.png                una caduta vista sui segnali inerziali
    caratteristiche_ricorsive.png     le quattro caratteristiche durante una caduta
    albero_primi_livelli.png          i primi due livelli dell'albero decisionale
    anticipo_modello.png              distribuzione dell'anticipo sulle cadute riconosciute
    anticipo_baseline.png             la stessa distribuzione per l'algoritmo a soglie
    compromesso_baseline.png          compromesso fra cadute riconosciute e falsi allarmi
    confronto_albero_baseline.png     confronto fra il modello e l'algoritmo di riferimento
    albero_mlc.txt                    l'albero nel formato importato dal sensore
    mlc.json                          la configurazione dei registri del sensore
    risultati_albero.csv              le dodici configurazioni di soglia e persistenza
    risultati_baseline_loso.csv       l'algoritmo di riferimento, stessa validazione
    baseline_griglia.csv              la griglia di soglie provate per il riferimento
    registrazioni_dispositivo.zip     le registrazioni fatte con il modello nel sensore

Il notebook è diviso in otto parti: verifica del dataset, algoritmo a soglie di
riferimento, caratteristiche ricorsive, tabelle di addestramento, albero decisionale,
validazione incrociata e confronto, esportazione verso il sensore, analisi delle
registrazioni fatte con il modello in esecuzione nel sensore. GitHub lo mostra
direttamente nel browser, con il codice e i grafici già eseguiti.

I file intermedi pesanti — la tabella di addestramento e i risultati grezzi della
validazione — non sono inclusi: li rigenera il notebook.

## Dati

Il dataset **KFall** non è incluso in questo repository e va scaricato dagli autori:

- sito del dataset: <https://sites.google.com/view/kfalldataset>
- articolo: X. Yu et al., *A Large-Scale Open Motion Dataset (KFall) and Benchmark
  Algorithms for Detecting Pre-impact Fall of the Elderly Using Wearable Inertial
  Sensors*, Frontiers in Aging Neuroscience, 2021, doi
  [10.3389/fnagi.2021.692865](https://doi.org/10.3389/fnagi.2021.692865)

KFall contiene 5.075 registrazioni di 32 giovani adulti (2.346 cadute simulate e 2.729
attività quotidiane), acquisite a 100 Hz con un sensore inerziale sulla zona lombare.
È stato scelto perché, per ogni caduta, annota sia l'istante in cui il movimento
comincia sia l'istante dell'impatto: senza questi due istanti si può verificare che una
caduta venga riconosciuta, ma non misurare con quanto anticipo.

Una volta scaricato, il dataset va messo così:

    dati/
      sensor_data/     le cartelle SA06 ... SA38 con i file .csv
      labels/          i file .xlsx con le etichette di ogni soggetto

e il percorso va indicato nella prima cella del notebook.

Le registrazioni fatte con il sensore sono invece incluse, in
`registrazioni_dispositivo.zip`: tre prove di caduta con lunghezze di finestra diverse
(`cadute_mlc2.csv`, `cadute_mlc3.csv`, `cadute_mlc4.csv`) e due registrazioni con il
dispositivo indossato (`indossato.csv`, `indossato2.csv`). Ogni file riporta
accelerazione, velocità angolare e l'uscita del Machine Learning Core, che vale 0 per
"non caduta" e 4 per "caduta".

## Come riprodurre

    git clone https://github.com/federicoperronescuola-creator/modello-previsione-caduta.git
    cd modello-previsione-caduta
    python -m venv .venv
    .venv\Scripts\activate        # su Windows
    pip install numpy pandas scipy matplotlib scikit-learn openpyxl jupyter
    jupyter notebook modello_caduta.ipynb

Le celle vanno eseguite in ordine. La validazione leave-one-subject-out ripete
l'addestramento 32 volte ed è la parte più lenta: qualche minuto su un portatile.
L'ottava parte non richiede KFall: lavora soltanto sui file di
`registrazioni_dispositivo.zip`.

## Dal modello al sensore

L'albero non viene eseguito da Python sul dispositivo: viene tradotto in una
configurazione del Machine Learning Core e scritto nei registri del sensore, che da quel
momento calcola le caratteristiche e valuta l'albero da solo, senza bisogno che il
microcontrollore sia acceso.

Il trasferimento è stato eseguito e provato sulla scheda STEVAL-MKBOXPRO
(SensorTile.box PRO), che monta il LSM6DSV16X. Due file di questo repository lo
documentano:

    albero_mlc.txt   l'albero nel formato testuale importato da MEMS Studio
    mlc.json         la configurazione dei registri generata per il sensore

Esito: l'albero occupa **119 nodi dei 128 disponibili** (59 di decisione e 60 foglie) e
impiega 4 delle 31 caratteristiche configurabili; la configurazione consiste in 338
scritture di registro. Caricata nel sensore, l'uscita del Machine Learning Core
(registro `MLC1_SRC`) passa da 0 a 4 durante la caduta, **prima dell'impatto**: su
quattro cadute simulate l'anticipo rispetto al picco di accelerazione è risultato di
269, 199, 277 e 225 ms, con mediana 247 ms. All'istante della decisione il modulo
dell'accelerazione era fra 0,10 e 0,17 g, cioè in piena caduta libera.

Il trasferimento ha richiesto tre adattamenti che la documentazione non rende espliciti,
e tutti e tre sono implementati nelle celle di esportazione del notebook:

- le caratteristiche ricorsive del sensore sono filtri IIR con parametri `b1`, `b2`,
  `a2`; il coefficiente α dell'addestramento corrisponde a `b1 = 1`, `b2 = α`,
  `a2 = 1 − α`;
- il sensore esprime la norma della velocità angolare in **rad/s**, mentre il modello è
  addestrato su valori in **dps**: le soglie sul giroscopio vanno divise per 180/π
  (quelle sulla varianza per il quadrato dello stesso fattore);
- la finestra del Machine Learning Core **non è scorrevole**: l'albero produce un
  risultato una volta per finestra e il filtro ricorsivo si aggiorna con la stessa
  cadenza. Con dodici campioni a 120 Hz questo significa una decisione ogni 100 ms.

## Cosa non torna fra calcolatore e dispositivo

Lo stesso modello, eseguito nel sensore e indossato, produce fra 22 e 27 segnalazioni al
minuto durante le normali attività quotidiane, contro le 3,0 al minuto che produce in
Python sulle attività quotidiane di KFall. Il confronto ristretto al solo cammino è
ancora più netto: 36–58 al minuto sul dispositivo contro 0,0–2,6 in Python.

Lo scarto non dipende da come sono state fatte le registrazioni. Il segnale registrato
indossando il dispositivo è **più tranquillo** di quello di KFall durante il cammino
(deviazione standard del modulo 0,09–0,11 g contro 0,17–0,27 g; picchi 1,5 g contro
2,9–3,4 g), e da fermo il sensore non produce quasi alcuna segnalazione. La causa sta
dunque nel trasferimento, cioè nelle caratteristiche che il sensore calcola, e la
spiegazione più probabile è l'aggiornamento del filtro una volta per finestra invece che
a ogni campione: la memoria effettiva del filtro passa da 100 ms a circa 1 s. Questa
misura però non isola il meccanismo, quindi resta l'ipotesi più compatibile con i dati,
non un fatto dimostrato.

Una mitigazione pratica c'è: chiedere al meta-classificatore due risultati consecutivi
prima di dare l'allarme porta le segnalazioni da 26,5 a 6,1 e da 22,2 a 3,2 al minuto
senza perdere nessuna delle quattro cadute, al prezzo di una finestra di ritardo.

Restano fuori da questo lavoro le misure di corrente e di autonomia, e il
riaddestramento del modello sulle caratteristiche così come il sensore le calcola, che è
l'intervento che aggredisce direttamente lo scarto appena descritto.

## Licenza

Codice rilasciato con licenza MIT. Il dataset KFall resta soggetto alle condizioni dei
suoi autori.
