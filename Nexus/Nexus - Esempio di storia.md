---
tags: [nexus, veil, lore, esempio]
---

# Nexus - Esempio di storia

> `free-text.txt`: il materiale di scrittura "La Tessitura del Velo", scritto a mano. È l'esempio di come si presenta una storia a Nexus.

## Perché è un esempio
Una storia si scrive **una volta**, in testo libero, senza schema: mondo, fazioni, temi, luoghi, personaggi con movente e obiettivo, catena di scambi, finale. Poi:

```
free-text.txt --(just compile, LLM)--> lore.txt --(just lore)--> ANTEFATTO + NPC_PREFIX.* + record NPC
                                                 --(just docs)--> pagine VEIL-*.md
```

Il risultato strutturato è in [[Nexus - Lore]]. Per un'altra storia basta un altro file: `just compile altra_storia.txt`.

## Cosa contiene un buon testo libero
- **Mondo**: premessa, conflitto, che cosa è in gioco (qui il Velo che "dimentica").
- **Fazioni** in conflitto vero, con un capo ciascuna (Lyra, Theron, Boros).
- **Temi** espliciti.
- **Un blocco per NPC**: dove vive, ruolo, movente, obiettivo, che cosa custodisce, come trattarlo, coordinate.
- **Missioni e scambi**: la catena di oggetti, chi dà che cosa in cambio di che cosa.
- **Finale**: condizioni per la scelta ultima e sua irreversibilità.

## Testo (free-text.txt, 2026-10-20)

```text
LA TESSITURA DEL VELO - materiali di scrittura (testo libero)

MONDO
Eldoria non è protetta da una semplice barriera magica. Il Velo è un tessuto vivente di memorie, sogni e sacrifici - l'ultima opera di una civiltà antica che scelse di dissolvere la propria esistenza per diventare protezione eterna.

Ogni filo del Velo custodisce i ricordi di chi fu: i Tessitori, una razza che padroneggiava l'arte di intrecciare realtà e memoria. Quando una minaccia cosmica - gli Oblivianti, entità che si nutrono di esistenza stessa - giunse nel loro mondo, i Tessitori non fuggirono né combatterono. Si sacrificarono volontariamente, trasformando le proprie anime in un intreccio protettivo che avvolge Eldoria come un bozzolo di sogni.

Per millenni il Velo ha funzionato perfettamente, filtrando non solo minacce fisiche ma preservando la memoria collettiva dell'umanità. Ogni storia raccontata, ogni canzone cantata, ogni atto d'amore o coraggio si intreccia nel tessuto, rafforzandolo.

Ma ora il Velo sta "dimenticando". Non si deteriora - si dimentica del proprio scopo. I fili più antichi, quelli che custodiscono il Sacrificio Originale, iniziano a sfilacciarsi perché nessuno più ricorda perché esiste il Velo. Gli abitanti di Eldoria lo danno per scontato, e questa dimenticanza collettiva lo sta lentamente distruggendo.

Dove si aprono le prime lacerazioni, filtrano non mostri o magia selvaggia, ma i Sussurri dell'Oblio - frammenti di nulla che cancellano ricordi, storie, persino l'idea stessa di esistere. Intere famiglie spariscono non fisicamente, ma dalla memoria. Libri si svuotano. Le tradizioni orali si interrompono a metà frase.

Tre fazioni si contendono la risposta alla crisi:

I CUSTODI DELLA MEMORIA credono che solo ricostruendo completamente la storia dei Tessitori e del loro sacrificio si possa rinnovare il Velo. Guidati dalla Tessitrice Lyra, cercano i Nodi Primordiali - luoghi dove la memoria del Sacrificio è ancora intatta.

I PROGRESSISTI, sotto l'influenza dell'Alto Giudice Theron, vedono il Velo come una reliquia del passato che ostacola l'evoluzione umana. Credono che si debba permettere al Velo di dissolversi e affrontare gli Oblivianti con tecnologia e forza di volontà moderni.

I VIANDANTI DELL'EQUILIBRIO, un ordine monastico guidato dall'anziano Boros, sostengono che il Velo non va né preservato né distrutto, ma trasformato - evoluto in qualcosa di nuovo che onori il passato senza imprigionare il futuro.

Il Cercatore si ritrova al centro di questo conflitto filosofico, dove ogni scelta non riguarda solo il destino di Eldoria, ma la natura stessa della memoria, dell'identità e del prezzo della protezione.

Temi:
- Il valore della memoria collettiva vs il diritto all'oblio
- Sacrificio intergenerazionale e gratitudine verso gli antenati
- Il paradosso della protezione: ciò che ci salva può anche imprigionarci
- L'importanza delle storie nella definizione dell'identità
- Il conflitto tra tradizione e progresso

LUOGHI

City (accesso: Controlled_Access). Una metropoli imponente con grattacieli che si stagliano contro il Velo. La City domina lo skyline mentre la città bassa si estende ai suoi piedi. Grandi Occhi di sorveglianza sorvegliano ogni angolo, ma esistono sempre zone d'ombra dove i ribelli si nascondono. Legame col Velo: La città rappresenta il tentativo dell'umanità di controllare e sfruttare il Velo per il potere. I documenti burocratici resistono misteriosamente ai Sussurri dell'Oblio, creando un archivio involontario di memoria. Collegamenti: [ "nexus_of_paths", "sanctum_of_whispers", "village_and_surrounding_areas", "underground_resistance_network" ]

Forest (accesso: Open). Una foresta antica e misteriosa dove alberi secolari si ergono come guardiani silenziosi. La luce del sole filtra attraverso le foglie creando giochi d'ombra che danzano sul terreno coperto di muschio. Il profumo di terra umida e resina si mescola con il mormorio del vento tra le fronde. Legame col Velo: La foresta è un luogo dove il Velo si manifesta attraverso i sussurri del vento e le ombre che sembrano muoversi di vita propria. Gli alberi più antichi custodiscono memorie di ere passate, e chi sa ascoltare può sentire gli echi delle storie dimenticate. Collegamenti: [ "village_outskirts", "ancient_ruins_entrance", "mountain_foothills", "tavern_back_path" ]

Liminal Void (accesso: Restricted_Advanced). Uno spazio esistenziale che sfida ogni comprensione fisica. Il vuoto liminale appare come un'estensione infinita di nebbia opalescente dove il concetto stesso di direzione perde significato. Qui la realtà è fluida, e frammenti di memorie perdute fluttuano come isole di luce in un mare di incertezza. Legame col Velo: Il Liminal Void è il cuore stesso del Velo, dove tutte le memorie dimenticate convergono prima di dissolversi nell'oblio. È qui che i frammenti di storie perdute cercano di ricomporsi, e dove chi è abbastanza coraggioso può tentare di recuperare ciò che sembrava perduto per sempre. Collegamenti: [ "nexus_of_paths_deep_access", "sanctum_of_whispers_hidden_passage", "ancient_ruins_secret_chamber", "void_entrance_points" ]

Mountain (accesso: Open_challenging). Imponenti vette rocciose che si ergono verso il cielo, con sentieri serpentini che si inerpicano tra dirupi e crepacci. L'aria è sottile e pura, e il vento trasporta echi di antiche leggende. Caverne nascoste e altipiani nascosti custodiscono segreti di epoche remote, mentre il panorama offre una vista mozzafiato su tutto il mondo sottostante. Legame col Velo: Le montagne sono pilastri che sostengono il Velo stesso, luoghi dove le memorie più profonde e durature trovano rifugio. Le cime più alte toccano quasi il tessuto della realtà, e chi raggiunge queste vette può percepire la vastità delle storie che il Velo contiene. Collegamenti: [ "forest_mountain_path", "ancient_ruins_highland_entrance", "village_mountain_trade_route", "sky_bridges_to_nexus" ]

Nexus of Paths (accesso: Restricted_Endgame). Un crocevia mistico dove tutti i sentieri di Eldoria convergono. Al centro si erge un antico cerchio di pietre coperte di rune, con Meridia che tesse al grande Telaio del Nuovo Inizio. L'aria vibra di potenziale e destino. Legame col Velo: Il Nexus è il punto di maggiore tensione del Velo, dove tutte le linee di forza convergono. È qui che la scelta finale del Cercastorie avrà effetto su tutto l'Yperverso. Collegamenti: [ "all_major_locations_converge_here", "sanctum_of_whispers_via_sacred_path", "city_via_political_route", "ancient_ruins_via_memory_path", "mountain_via_wisdom_trail" ]

Rovine Antiche (accesso: Open_dangerous). Resti maestosi di una civiltà perduta, dove colonne spezzate e archi crollati testimoniano una grandezza dimenticata. Edera e muschio ricoprono le antiche sculture, mentre stanze segrete si aprono nei sotterranei. L'aria è densa di mistero e il suono del vento tra le rovine sembra sussurrare segreti di epoche remote. Legame col Velo: Le Rovine Antiche sono un epicentro di memorie stratificate, dove millenni di storia si sono sedimentati nel Velo. Ogni pietra è impregnata di ricordi di una civiltà che comprendeva profondamente la natura del Velo stesso. Chi esplora questi luoghi può accedere a frammenti di conoscenza ancestrale. Collegamenti: [ "ancient_ruins_main_site", "forest_ruins_pathway", "mountain_ancient_observatory", "underground_passages_to_city" ]

Sanctum of Whispers (accesso: Open_but_Reverent). Un antico santuario costruito con pietre bianche che sembrano brillare di luce propria. Grandi archi si aprono verso il cielo, e il vento che passa attraverso le colonne produce suoni melodici. Al centro si trova un altare tessitore coperto di simboli antichi. Legame col Velo: Il Sanctum è uno dei punti dove il Velo è più sottile. Qui Lyra può sentire direttamente i Sussurri dell'Oblio e comunicare con i Tessitori morti attraverso il tessuto stesso del Velo. Collegamenti: [ "ancient_ruins_syra", "nexus_of_paths_meridia", "mountain_boros", "pathways_to_all_areas" ]

Tavern (accesso: Public). Una taverna accogliente con tavoli di legno scuro, un focolare sempre acceso, e scaffali pieni di bottiglie dai riflessi dorati. L'atmosfera è calda e invitante, con il profumo di cibo speziato e il mormorio di conversazioni. Legame col Velo: La taverna si trova su un punto di flusso delle memorie del Velo. I viaggiatori qui spesso hanno sogni vividi che sono echi di ricordi tessuti nel Velo stesso. Collegamenti: [ "village_garin_forge", "ancient_ruins_syra", "paths_to_city", "forest_edge" ]

Village (accesso: Public). Un pittoresco villaggio con case di pietra e legno dai tetti di paglia, dove il fumo sale placido dai camini e i giardini fioriti colorano le strade acciottolate. La vita scorre tranquilla tra botteghe artigiane, mercati locali e la piazza centrale dove i bambini giocano e gli anziani condividono storie al tramonto. Legame col Velo: Il villaggio è costruito su un nodo di memorie quotidiane, dove i ricordi semplici della vita di tutti i giorni si intrecciano nel Velo. Le tradizioni familiari e le storie tramandate di generazione in generazione creano un tessuto denso di memorie condivise che rafforzano la coesione della comunità. Collegamenti: [ "tavern_village_center", "forest_village_outskirts", "mountain_trade_routes", "paths_to_city" ]

PERSONAGGI

Syra vive a Ancient Ruins. Ruolo: Eco Residuo, Tessitrice Incompleta. Movente: Completare la propria trasformazione nel Velo, ma solo se il Cercatore dimostra di comprendere veramente cosa significa sacrificare la propria esistenza per altri. Obiettivo: Trasferire al Cercatore la propria esperienza del momento di trasformazione, fornendo il Cristallo di Memoria Antica di cui Lyra ha bisogno. Custodisce: L'Ultimo Respiro - la memoria dell'ultimo momento prima che i Tessitori si dissolvessero nel Velo, contenente l'emozione pura del loro sacrificio Come trattarlo: Syra non vuole pietà o promesse. Vuole che il Cercatore comprenda il peso emotivo del sacrificio. Solo chi è disposto a sacrificare qualcosa di veramente proprio può ricevere la sua memoria. Coordinate: 153,78,26.

Cassian vive a City. Ruolo: Burocrate Opportunista, Mercante di Influenze. Movente: Mantenere la propria posizione di potere indipendentemente da chi vinca nel conflitto filosofico sul Velo. Obiettivo: Ottenere informazioni compromettenti su tutti i leader delle fazioni per poter negoziare con chiunque risulti vincitore. Custodisce: Archivio delle Debolezze - dossier dettagliati sui punti deboli di tutte le figure importanti di Eldoria Come trattarlo: Cassian è completamente mercenario ma non stupido. Se il Cercatore gli dimostra di avere potere reale, sarà un alleato prezioso. Se lo tratta con disprezzo, diventerà un nemico subdolo ma pericoloso. Coordinate: 363,162,24.

Irenna vive a City. Ruolo: Marionettista della Resistenza, Artista della Memoria Vivente. Movente: Usare il teatro di marionette per preservare storie che i Sussurri dell'Oblio stanno cercando di cancellare. Obiettivo: Creare una rappresentazione teatrale che "fissi" nella memoria collettiva la verità sui crimini di Theron e sui pericoli della sua filosofia. Custodisce: Fili dell'Intreccio - corde di marionetta intessute con frammenti del Velo stesso, che permettono di "animare" ricordi e farli rivivere teatralmente Come trattarlo: Irenna è un'artista prima che una rivoluzionaria. La convince di più una storia ben raccontata che un argomento logico. Vuole emozione e verità umana, non prove legali. Coordinate: 385,191,23.

Alto Giudice Theron vive a City. Ruolo: Leader dei Progressisti, Rivoluzionario Filosofico. Movente: Liberare l'umanità dalla dipendenza dal Velo e dai fantasmi del passato, credendo che solo affrontando direttamente gli Oblivianti l'umanità potrà evolversi veramente. Obiettivo: Convincere il Cercatore che il Velo è una prigione dorata e che la vera forza sta nell'accettare l'oblio e ricostruire l'identità da zero, senza memorie che limitino le possibilità. Custodisce: Codice della Tabula Rasa - una tecnica filosofica e magica per cancellare selettivamente memorie dolorose e limitanti, mantenendo solo ciò che serve per il futuro Come trattarlo: Theron non è malvagio - è genuinamente convinto che la sua via sia migliore. Sfida il Cercatore a questionare se preservare ogni memoria, anche quella del dolore e del trauma, sia davvero un bene. La sua "corruzione" è filosofica, non magica. Coordinate: 390,165,24.

Elira vive a Forest. Ruolo: Custode del Nodo Naturale, Interprete dei Sussurri. Movente: Proteggere il Nodo Primordiale nascosto nella foresta, dove le memorie dei Tessitori si intrecciano con la vita naturale. Obiettivo: Valutare se il Cercatore è degno di accedere al Nodo, testando la sua capacità di ascoltare la natura senza imporle la propria volontà. Custodisce: Seme della Memoria - un frammento del primo albero che crebbe dove un Tessitore si dissolse nel Velo, contenente la pace di chi accettò il sacrificio Come trattarlo: Elira non accetta parole o promesse. Vuole che il Cercatore dimostri empatia reale verso gli esseri viventi che non possono parlare per difendersi. Il test è sottile e basato su azioni, non su risposte corrette. Coordinate: 259,182,22.

Erasmus vive a Liminal Void. Ruolo: Sussurro Consapevole, Ambasciatore dell'Oblio. Movente: Offrire al Cercatore la prospettiva degli Oblivianti - non sono malvagi, ma rappresentano un processo naturale di rinnovamento cosmico. Obiettivo: Convincere il Cercatore che l'oblio non è morte ma trasformazione, e che opporsi agli Oblivianti è come opporsi al cambiamento delle stagioni. Custodisce: Visione del Vuoto Fertile - l'esperienza diretta di cosa significa essere parte del ciclo cosmico dell'oblio e della rinascita Come trattarlo: Erasmus è l'unico NPC che non vuole nulla dal Cercatore. Offre solo una prospettiva diversa, inquietante ma sincera. Parlare con lui è facoltativo ma potrebbe cambiare radicalmente la comprensione del Cercatore su cosa sta realmente accadendo. Coordinate: 2,3,27.

Boros vive a Mountain. Ruolo: Primo Monaco dell'Equilibrio, Testimone delle Ere. Movente: Trovare una terza via che onori il passato senza imprigionare il futuro, trasformando il Velo in qualcosa di nuovo che serva l'umanità di oggi. Obiettivo: Insegnare al Cercatore la Filosofia del Flusso - che tutto deve cambiare per rimanere vivo, incluso il Velo stesso. Custodisce: Il Bastone del Viandante - un artefatto che rivela tutte le possibili conseguenze di una scelta, permettendo di vedere i futuri che si diramano da ogni decisione Come trattarlo: Boros non offre risposte semplici. Mette alla prova la saggezza del Cercatore attraverso paradossi e domande che non hanno risposta "giusta". Vuole essere sicuro che il Cercatore possa pensare oltre le ideologie di Lyra e Theron. Coordinate: 437,395,60.

Meridia vive a Nexus of Paths. Ruolo: Tessitrice del Destino, Guardiana delle Scelte Finali. Movente: Assicurarsi che la scelta finale del Cercatore sia presa con piena consapevolezza delle conseguenze, non per impulso o manipolazione. Obiettivo: Preparare il Cercatore al momento della Scelta Ultima, quando dovrà decidere il destino del Velo davanti ai rappresentanti di tutte le fazioni. Custodisce: Il Telaio del Nuovo Inizio - lo strumento che permetterà di attuare qualsiasi decisione prenda il Cercatore, che sia riparare, distruggere o trasformare il Velo Come trattarlo: Meridia è la fine del gioco. Non apparirà fino a quando il Cercatore non avrà interagito significativamente con tutte e tre le fazioni principali. La sua "quest" è essenzialmente il finale del gioco, dove tutte le scelte precedenti convergono. Coordinate: 300,257,24.

Lyra vive a Sanctum of Whispers. Ruolo: Tessitrice Suprema, Ultima Custode della Memoria Primordiale. Movente: Preservare e ricostruire la conoscenza dei Tessitori per rinnovare il Velo prima che i Sussurri dell'Oblio consumino tutto ciò che esiste. Obiettivo: Guidare il Cercatore nella raccolta dei Frammenti di Memoria dai Nodi Primordiali per tessere un nuovo Atto di Ricordo che possa risvegliare il Velo alla sua vera natura. Custodisce: Il Telaio dell'Eco - un artefatto che permette di "leggere" le memorie intessute nel Velo e di comunicare con i Tessitori morti Come trattarlo: Lyra non vuole semplici oggetti o prove. Vuole che il Cercatore *viva* l'esperienza della memoria, visitando i luoghi dove il Sacrificio Originale risuona ancora. Solo chi comprende veramente il peso del sacrificio può tessere un nuovo Atto di Ricordo. Coordinate: 411,281,41.

Jorin vive a Tavern. Ruolo: Raccoglitore di Sogni Perduti, Cronista Inconsapevole. Movente: Collezionare e preservare i sogni e le storie che i Sussurri dell'Oblio stanno cancellando, senza rendersi completamente conto del loro valore. Obiettivo: Completare la sua raccolta di "Sogni Impossibili" - storie che la gente sogna ma che non corrispondono a nessuna memoria reale. Custodisce: Diario delle Memorie Sospese - una collezione di sogni e storie che in realtà sono frammenti di memorie cancellate dai Sussurri, preziose per ricostruire ciò che è andato perduto Come trattarlo: Jorin non sa di essere importante. Trattalo come un semplice locandiere curioso, ma ascolta attentamente le storie che racconta - potrebbero contenere indizi cruciali su cosa sta davvero succedendo al Velo. Coordinate: 108,228,29.

Garin vive a Village. Ruolo: Fabbro delle Memorie, Artigiano dell'Impossibile. Movente: Forgiare oggetti che "ricordano" le loro origini e i loro scopi, resistendo così ai Sussurri dell'Oblio. Obiettivo: Creare l'Ancora della Persistenza - un'arma o strumento che non può essere dimenticato una volta forgiato. Custodisce: Lama della Memoria Incisa - una spada che "ricorda" ogni battaglia combattuta e ogni vita salvata, diventando più forte con l'uso altruistico Come trattarlo: Garin valuta le persone dalla passione che mettono nei loro scopi. Non gli interessa se sei buono o cattivo, ma se credi davvero in ciò che fai. La sua ricompensa riflette l'intensità delle emozioni che il Cercatore porta nel loro rapporto. Coordinate: 213,303,27.

Mara vive a Village. Ruolo: Erborista della Memoria, Curatrice delle Radici Perdute. Movente: Preservare la conoscenza erboristica che si sta perdendo man mano che le piante "dimenticate" dai Sussurri dell'Oblio scompaiono dalla realtà. Obiettivo: Creare un elisir che possa proteggere temporaneamente dalla perdita di memoria causata dai Sussurri dell'Oblio. Custodisce: Infuso della Persistenza - una pozione che per alcune ore rende immuni agli effetti minori dei Sussurri dell'Oblio Come trattarlo: Mara è pragmatica e orientata alla sopravvivenza. Non le interessano le grandi filosofie, ma gli strumenti concreti per proteggere le persone che ama. La sua ricompensa è estremamente utile ma temporanea. Coordinate: 265,294,22.

MISSIONI E SCAMBI
La catena: Mara vende la Pozione di Guarigione per 50 crediti. Elira da il Seme della Foresta in cambio di una Pozione di Guarigione. Boros da il Minerale di Ferro Antico in cambio di un Seme della Foresta. Garin da i Trucioli di Ferro in cambio di un Minerale di Ferro Antico. Jorin da la Ciotola dell'Offerta Sacra in cambio di Trucioli di Ferro. Syra da il Cristallo di Memoria Antica in cambio di una Ciotola dell'Offerta Sacra, e il Cristallo serve a Lyra. Cassian, fuori dalla catena, vende la Pergamena della Saggezza per 50 crediti. L'oggetto passa solo dopo consegna o pagamento.

IL FINALE
Meridia non si mostra finche il Cercatore non ha parlato davvero (non solo scambiato oggetti) con i tre capi fazione: Lyra (Custodi della Memoria, preservare il Velo), Theron (Progressisti, lasciarlo dissolvere) e Boros (Viandanti dell'Equilibrio, trasformarlo), almeno tre turni di dialogo ciascuno. Poi il Cercatore sceglie con il comando /choose preserve, dissolve o transform. La scelta e irreversibile. Meridia non sceglie per lui, non lo spinge e non giudica: espone le tre vie con il loro prezzo. Il suo oggetto e Il Telaio del Nuovo Inizio, che attua qualsiasi decisione. Il giocatore e il Cercatore (mai Cercastorie).

```

Progetto: [[nexus]] · Vedi anche: [[Nexus - Lore]]
