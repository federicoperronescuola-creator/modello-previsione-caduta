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

## Le quattro caratteristiche

Il MLC non ha memoria per conservare una finestra di segnale, quindi le caratteristiche
sono calcolate in forma **ricorsiva**: ogni nuovo campione aggiorna il valore precedente
senza che il passato venga conservato.

    y[i] = ALFA * x[i] + (1 - ALFA) * y[i-1]      ALFA = 0,10

Le quattro grandezze usate dall'albero sono la media ricorsiva e la varianza ricorsiva
del modulo dell'accelerazione e della velocità angolare. Restano piatte durante le
attività quotidiane e si muovono con decisione appena la caduta comincia, e questo è
ciò che permette di decidere **a ogni campione** invece di aspettare una finestra intera.

## Struttura del repository

    codice/
      modello_caduta.ipynb      l'intero lavoro, dal dataset al modello finale
    risultati/
      baseline_griglia.csv      algoritmo a soglie, griglia delle combinazioni provate
      risultati_baseline_loso.csv  algoritmo a soglie, validazione leave-one-subject-out
      risultati_albero.csv      albero decisionale, le dodici configurazioni valutate
    figure/                     i grafici prodotti dal notebook
    requirements.txt

Il notebook è diviso in sette parti: verifica del dataset, algoritmo a soglie di
riferimento, caratteristiche ricorsive, tabelle di addestramento, albero decisionale,
validazione incrociata e confronto, esportazione verso il sensore.

I file intermedi pesanti (`tabella_addestramento.csv`, `risultati_v.pkl`) non sono
inclusi: li rigenera il notebook.

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

## Come riprodurre

    git clone <url-di-questo-repository>
    cd <cartella>
    python -m venv .venv
    .venv\Scripts\activate        # su Windows
    pip install -r requirements.txt
    jupyter notebook codice/modello_caduta.ipynb

Le celle vanno eseguite in ordine. La validazione leave-one-subject-out ripete
l'addestramento 32 volte ed è la parte più lenta: qualche minuto su un portatile.

## Dal modello al sensore

L'albero non viene eseguito da Python sul dispositivo: viene tradotto in una
configurazione del Machine Learning Core e scritto nei registri del sensore, che da quel
momento calcola le caratteristiche e valuta l'albero da solo, con un assorbimento
dell'ordine dei microampere e senza bisogno che il microcontrollore sia acceso.

Il percorso è: albero addestrato in Python → file ARFF con le caratteristiche →
generazione dell'albero in MEMS Studio → configurazione del sensore → prova sulla
scheda STEVAL-MKBOXPRO (SensorTile.box PRO).

## Licenza

Codice rilasciato con licenza MIT. Il dataset KFall resta soggetto alle condizioni dei
suoi autori.
