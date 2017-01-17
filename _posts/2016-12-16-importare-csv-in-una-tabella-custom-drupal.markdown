---
layout:   post
title:    "Immagazzinare un file CSV nel database di Drupal"
subtitle: "Come importare i dati di un file CSV dentro ad una tabella custom in Drupal."
date:     2016-12-16 13:11:00
author:   "Roberto Peruzzo"
tags:     drupal7, csv, importing
comments: true
---

<p>
  In questo post vedremo come prendere i dati da un file <em>CSV</em> e salvarli dentro
  una tabella creata ad-hoc all'interno del database della nostra <em>web application</em>
  fatta in <strong>Drupal</strong>. Per la lettura e l'importazione dei dati utilizzando lo
  <em>statement MySQL</em> <code>LOAD DATA INFILE</code> che ci permette di fare tutto in un colpo solo.
</p>
<p>
  In questo mio caso non devo importare delle
  <a href="https://www.drupal.org/docs/7/api/entity-api/an-introduction-to-entities">
  <em>entità di Drupal</em></a>, come
  ad esempio contenuti (node), utenti, tassonomie o file, ma un sacco di dati
  su da elaborare per poi utilizzarli per la costruzione di alcuni grafici
  riepilogativi. Quindi non ho scelto di utilizzare uno tra metodi classici
  per importare contenuti in Drupal, come
  <a href="https://www.drupal.org/project/feed_import"><em>Feeds Import</em></a> o
  <a href="https://www.drupal.org/project/migrate"><em>Migrate</em></a>.
  Trattandosi inoltre di file CSV piuttosto grossi (~80MB) con migliaia di righe,
  ho preferito utilizzare una mia tabella con la struttura più adatta a:
</p>
<ul>
  <li>
    <strong>veloccizzare l'importazione dei dati</strong>, grazie allo <em>statement</em>
    <code>LOAD DATA INFILE</code> è possibile importare 13.733.412 (312.123 righe
    per 44 colonne) valori in circa 10 secondi, senza effettuare ancora alcun
    tipo di ottimizzazione a MySQL e lavorando in locale sul mio MacBook Pro;
  </li>
  <li>
    <strong>velocizzare il recupero dei dati</strong>, ottimizzando la tabella
    con chiavi e indici personalizzati. Nel caso avessi deciso invece di
    definire un mio <a href="https://www.drupal.org/docs/7/api/entity-api/an-introduction-to-entities">
    <em>Bundle</em></a>, mi sarei trovato ad avere 44 tabelle del tipo <code>field_data_field_*</code>
    ogniuna relativa ad una colonna del file CSV e messa in relazione con la
    tabella <code>node</code>, aumentando così la complessità della query SQL
    per recuperare una serie di dati.
  </li>
</ul>

<p>Vediamo qui di seguito i passaggi principali per realizzare l'importazione.</p>

<h2 class="section-heading">Creazione del modulo</h2>
<p>
  Per prima mi sono creato il mio modulo. Per chi non l'avesse mai fatto può
  seguire l'utile guida <a href="https://www.drupal.org/docs/7/creating-custom-modules">
  <em>Creating custom modules</em></a>. Per velocizzare questo passaggio potete
  utilizzare il codice che mi son preparato nel mio <a href="https://github.com/robertoperuzzo/drupal7_base_module">repository di GitHub</a>.
  Una volta che ve lo siete clonato, dategli il nome che più preferite.
  Nel mio caso l'ho chiamato <code>import_csv_data</code>.
</p>

<h2 class="section-heading">Definizione della tabella</h2>
<p>
  Ora che abbiamo il nostro <em>custom module</em> pronto, possiamo definire lo schema
  della nostra tabella in cui andremo a salvare i dati presenti nel file CSV.
  Per farlo utilizzeremo il <em>gancio</em>
  <a href="https://api.drupal.org/api/drupal/modules%21system%21system.api.php/function/hook_schema/7.x">
    <code>hook_schema()</code></a> e lo inserisco nel mio file <code>import_csv_data.install</code>.
  Questo <em>hook</em> permette di creare la tabella alla prima installazione del modulo
  e di eliminarla quando verrà disinstallato. Lo schema della tabella viene
  definito da un array associativo simile al seguente:
</p>
<pre>
  $schema['import_csv_data'] = array(
    'description' => 'The base table for noise measuring data.',
    'fields' => array(
      'nvid' => array(
        'description' => 'The primary identifier for a single measure.',
        'type' => 'serial',
        'unsigned' => TRUE,
        'not null' => TRUE,
        ),
      'fid' => array(
        'description' => 'The file id from which all data was loaded.',
        'type' => 'int',
        'unsigned' => TRUE,
        'not null' => TRUE,
        'default' => 0,
        ),
      'measuring_number' => array(
        'type' => 'int',
        'unsigned' => TRUE,
        'not null' => TRUE,
        'default' => 0,
        ),
      'measuring_datetime' => array(
        'description' => 'The Unix timestamp when the measure was done.',
        'type' => 'int',
        'unsigned' => TRUE,
        'not null' => TRUE,
        'default' => 0,
        ),
      'ch1slm_p1a_lapeakthdb' => array(
        'type' => 'float',
        'not null' => TRUE,
        'default' => 0,
        ),
      'indexes' => array(
      'measuring_datetime'  => array('measuring_datetime'),
      'fid'                 => array('fid'),
    ),
    'primary key' => array('nvid'),
  );
</pre>
<p>
  Il nome della tabella viene definito da nome del primo indice di questo array
  <code>$schema['import_csv_data']</code>; solitamente viene utilizzato
  lo stesso nome del modulo che la definisce.
  Allo stesso modo, anche il nome dei campi è definito dal nome dell'indice
  utilizzato nell'array; ad esempio per definire il campo di tipo <code>float</code>
  con nome <code>ch1slm_p1a_lapeakthdb</code> basterà scrivere il seguente array
</p>
<pre>
  $schema['import_csv_data']['fields']['ch1slm_p1a_lapeakthdb'] = array(
    'type' => 'float',
    'not null' => TRUE,
    'default' => 0,
  );
</pre>
<p>
  Per maggiori informazioni sulla struttura di questo array, sui tipi di dato
  utilizzabili e su come definire indici/chiavi, vi invito a
  <a href="https://www.drupal.org/node/146843">leggere questa guida</a>.
</p>

<h2 class="section-heading">Query SQL</h2>
<p>
  Ora che abbiamo la struttura della tabella di destinazione pronta, vediamo
  come popolarla con i dati contenuti nel nostro file CSV.
  Il problema principale di solito è riuscire a mappare correttamente i valori
  nelle colonne corrette e ignorare eventuali righe inutili messe in testa
  al file.
</p>
<p>
  Per farvi capire meglio di cosa sto parlando, vi mostro come appare il mio
  file csv.
</p>
<p>
  <img src="{{ site.baseurl }}/img/2016-12-16/csv_file.png" alt="Immagine contenuto del file csv">
  <span class="caption text-muted">Frammento del file CSV da importare.</span>
</p>
<p>
  Come potete vedere le celle con i dati da utilizzare sono leggermente
  spostati rispetto la cella iniziale in alto a sinistra (A1); quindi dobbiamo
  dire alla query SQL esattamente da quale riga iniziare a leggere e
  quali sono le colonne da utilizzare.
</p>
<p>
  Grazie all'opzione <code>IGNORE</code> <i>number</i> <code>LINES</code> identifichiamo
  quante righe possiamo ignorare dall'inizio del file prima di incontrare
  i dati utili. Nel nostro caso ne dobbiamo saltare 7, come si può vedere nel
  frammento di codice qui sotto.
</p>
<pre>
  LOAD DATA LOCAL INFILE '/path/to/file.csv' INTO TABLE import_csv_data
    FIELDS TERMINATED BY ','
    IGNORE 7 LINES
    (@col1,@col2,@col3,@col4,@col5)
    SET fid = ". $fid . ",
        measuring_number = @col3,
        measuring_datetime = UNIX_TIMESTAMP(STR_TO_DATE(@col4,'%d/%m/%Y %h:%i:%s')),
        ch1slm_p1a_lapeakthdb = @col5;
</pre>
<p>
  Per ignorare invece le prime due colonne e convertire la data della quarta colonna
  in TIMESTAMP, utilizzeremo una serie di variabili e la clausola SET.
  Le variabili <code>(@col1,@col2,@col3,@col4,@col5)</code> assumono di riga in riga
  i valori delle rispettive colonne a partire dalla prima fino alla quinta.
  Grazie alla clausola SET possiamo assegnare le variabili ai campi della nostra
  tabella mappandone così correttamente i valori ed eventualmente manipolarli,
  come nel caso della data, prima di essere salvati nel database.
</p>
<p>
  Per maggiori informazioni sulla sintassi di questa query, potete approfondire
  la vostra conoscenza nel <a href="http://dev.mysql.com/doc/refman/5.7/en/load-data.html">manuale di MySQL</a>.
</p>
<h2 class="section-heading">Caricare i dati</h2>
<p>
  Nel mio caso ho definito un
  <em>
    <a href="https://www.drupal.org/docs/7/api/entity-api/an-introduction-to-entities">
      Drupal Bundle
    </a>
  </em> (volgarmente detti anche <em>content-type</em>) di nome <em>Rilevazione</em>
  con un campo di tipo file dove carico il file CSV "zippato".
  Ogni volta che salvo/aggiorno un'istanza di <em>Rilevazione</em>, importo i
  dati dal file CSV nella mia tabella. Ecco di seguito la mia implementazione
  del <em>gancio</em> <a href="https://api.drupal.org/api/drupal/modules%21node%21node.api.php/function/hook_node_update/7.x">hook_node_update()</a>.
</p>
<pre>
  /**
   * Implements hook_node_update().
   */
  function noise_vibration_data_node_update($node) {

    // Import data from CSV file saved into field_measure_file.
    // This field located inside rivelazione bundle.
    if ($node->type == 'rilevazione') {
      // Retrieve file id from node field.
      $field_items = field_get_items('node', $node, 'field_measure_file');
      if ($field_items) {
        $field_measure_file_fid = reset($field_items)['fid'];
        $file = file_load($field_measure_file_fid);

        // Extract zip file.
        $zip = new ZipArchive;
        $file_realpath = drupal_realpath($file->uri);
        $result = $zip->open($file_realpath);
        $destination_dir = file_directory_temp();
        if ($result === TRUE) {
          $zip->extractTo($destination_dir);
          $dest_filename = $zip->getNameIndex(0);
          $zip->close();
          watchdog('noise_vibration_data', 'File @file extracted into @tmp', array(
            '@file' => $file_realpath,
            '@tmp' => $destination_dir,
            ));
        }
        else {
          watchdog('noise_vibration_data', 'Open file @file failed, code: @code',
            array(
              '@file' => $file_realpath,
              '@code' => $result,
              ),
            WATCHDOG_ERROR);
        }

        // Load file data into db.
        $result = populate_mysql_table_with_infile('noise_vibration_data',
          $destination_dir . '/'. $dest_filename,
          $field_measure_file_fid);

      }
    }
  }
</pre>
<p>
  Per importare i dati nel database ho definito una mia funzione
  <code>populate_mysql_table_with_infile()</code>. Come potete vedere nel codice
  seguente <b>non ho utilizzato</b> la funzione
  <code>
    <a href="https://api.drupal.org/api/drupal/includes%21database%21database.inc/function/db_query/7.x">
      db_query()
    </a>
  </code>, ma ho utilizzato un nuovo oggetto <code>PDO</code>. Questa scelta
  è legata al fatto che l'utilizzo delle variabili SQL <code>(@col1,@col2,@col3,@col4,@col5)</code>
  nella query, si crea un conflitto tra queste variabili legate allo <em>statement</em>
  <code>LOAD DATA INFILE</code> e gli eventuali definizione di argomenti
  nella funzione <code>db_query()</code>, come in questo semplice esempio:
  <pre>
    $node_title = db_query(
      'SELECT title FROM {node} WHERE nid = @nid',
      array('@nid' => $nid))->fetchField();
  </pre>
</p>
<p>Ecco la definizione completa della funzione.</p>
<pre>
  function populate_mysql_table_with_infile($table_name, $file_realpath, $fid) {
    $database = Database::getConnectionInfo()['default'];
    $data_source = 'mysql:host=' . $database['host'] . ';dbname=' . $database['database'];
    $db_user = $database['username'];
    $db_password = $database['password'];

    $connection = new PDO($data_source, $db_user, $db_password,
      array(
        PDO::ATTR_EMULATE_PREPARES => TRUE,
        PDO::MYSQL_ATTR_USE_BUFFERED_QUERY => TRUE,
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_PERSISTENT
      )
    );

    $query = "LOAD DATA LOCAL INFILE '" . $file_realpath . "' INTO TABLE " . $table_name . "
        FIELDS TERMINATED BY ','
        IGNORE 7 LINES
        (@col1,@col2,@col3,@col4,@col5)
        SET fid = ". $fid . ",
            measuring_number = @col3,
            measuring_datetime = UNIX_TIMESTAMP(STR_TO_DATE(@col4,'%d/%m/%Y %h:%i:%s')),
            ch1slm_p1a_lapeakthdb = @col5";

    $statement = $connection->prepare($query);
    $result = $statement->execute();
    $statement->closeCursor();

    return $result;
  }
</pre>
<p>
  Questo è tutto. Non esitate ad aggiungere il vostro commento qui sotto nel caso
  abbiate curiosità da chiedere, dirmi il vostro punto di vista o fornirmi
  altre soluzioni.
</p>
