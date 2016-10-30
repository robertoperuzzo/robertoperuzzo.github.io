---
layout:   post
title:    "Multitheading con Drush"
subtitle: "Come aggiornare il path alias di oltre 200K nodi in meno di 30min."
date:     2016-10-22 14:05:00
tags:     drupal7 multithread concurrent drush pathauto_node_update_alias
---

L'anno scorso mi sono trovato di fronte ad un problema alquanto scomodo:
<strong>ristrutturare gli URL</strong> dei contenuti presenti nel sito di un cliente per
<strong>migliorarne la leggibilità e l'indicizzazione</strong> nei motori di ricerca.
Tralasciando tutte le questioni inerenti ai redirect 301 da gestire, voglio concentrare
questo post sulle performance raggiunte per <strong>aggiornare più di 200.000 path alias</strong>.

Il mio primo approccio è stato quello di utilizzare il <em>"gancio"</em> <code>hook_update_N()</code>
come ho <a href="{{ '/2016/10/07/aggiornare-in-massa-l-urlalias-dei-contenuti' | prepend: site.baseurl }}" title="Aggiornare in massa l'urlalias dei contenuti.">descritto nell'articolo</a> di qualche settimana fa.
La procedura faceva correttamente il suo dovere, ma con tempi di esecuzione
inproponibili al cliente, poiché tenere <em>offline</em> per due ore e mezza un sito che
fa circa 6,5K di visite al giorno non lo aiuta a mantenere una buona reputazione nel web.

La prima cosa che mi è balzata in mente è stata quella di suddividere l'intera
mole di dati in porzioni più piccole da processare in modo concorrente: il
cosiddetto <strong>multithreading</strong>. L'
<a href="https://www.deeson.co.uk/labs/multi-processing-part-1-how-make-drush-rush" title="Multi Processing Part 1: How to make Drush rush" target="_blank">articolo di John Ennew</a> (Techical Lead in Deeson) è
stato provvidenziale, poiché mi è bastato personalizzare il suo <em>drush multithread manager</em>
per raggiungere il mioscopo. Per vostra conoscenza, potete trovare il
<a href="https://github.com/johnennewdeeson/drush-multi-processing" title="Concurrent Applications with Drush" target="_blank">codice completo</a> scritto da John su GitHub.

Vediamo allora come implementare l'aggiornamento dei <em>path alias</em> con un
comando <em>Drush multithreading</em>.

<h2 class="section-heading">Creare il comando Drush</h2>
<p>
  La prima cosa che ho fatto è stata quella di
  <a href="https://www.sitepoint.com/drupal-create-drush-command/" title="Drupal: How to Create Your Own Drush Command" target="_blank">creare il mio comando Drush</a>. Per essere più precisi
  ho creato due comandi:
</p>
<ul>
  <li>
    <code>update-alias</code>, che esegue l'aggiornamento del path alias per l'insieme di nodi
    che gli verrà passato dal gestore del multithread.
  </li>
  <li>
    <code>mt-command</code>, che eseguirà il comando <code>update-alias</code> in modalità
    multithread;
  </li>
</ul>
{% highlight php %}
  /**
   * Implements of hook_drush_command().
   */
  function my_module_drush_command() {
    $items = array();

    $items['mt-command'] = array(
      'description' => 'This command manages all multithread processes.',
      'arguments' => array(
        'limit' => 'Total number of jobs to up - use 0 for all.',
        'batch_size' => 'Number of jobs each thread will work on.',
        'threads' => 'Number of threads',
      ),
      'options' => array(
        'offset' => 'A starting offset should you want to start 1000 records in',
      ),
      'bootstrap' => DRUSH_BOOTSTRAP_DRUPAL_ROOT,
    );

    $items['update-alias'] = array(
      'description' => 'Update the node url alias.',
      'arguments' => array(
        'name' => 'The name of this process, this will be the thread id',
        'limit' => 'Number of jobs each thread will work on.',
        'offset' => 'A starting offset should you want to start 1000 records in',
      ),
      'bootstrap' => DRUSH_BOOTSTRAP_DRUPAL_ROOT,
    );

    return $items;
  }
{% endhighlight %}

<p>
  Vediamo in dettaglio il contenuto delle funzioni eseguite da questi due comandi.
  A proposito, per associare un conmando ad una funzione basta il nome di quest'ultima
  sia formato dal nome del comando sostituendo il 'meno' (-, hyphen) con il
  'trattino basso' (_, underscore) ed aggiungere come prefisso <code>drush_</code>.
  Quindi se il comando è <code>mt-command</code> la funzione sarà <code>drush_mt_command</code>.
</p>

<h2 class="section-heading">Aggiornare i path alias</h2>
<p>
  Partiamo dall funzione più semplice, quella che esegue l'aggiornamento degli URL.
</p>
{% highlight php %}
  /**
   * Updates url alias.
   */
  function drush_update_alias($thread, $limit, $offset, $node_types = '') {
    # code
  }
{% endhighlight %}

<p>
  Per prima cosa recuperiamo l'insieme di nodi da aggiornare. La cardinalità
  di questo insieme è definita dal paramentro <code>$limit</code>, mentre
  il punto di partenza da cui iniziare l'aggiornamento lo troviamo nel parametro
  <code>$offset</code>. Il parametro <code>$thread</code> infine contiene
  l'identificativo del thread in esecuzione su questo insieme di dati.
</p>
{% highlight php %}
  // Retrieve the next group of nids.
  $query = db_select('node', 'n');

  $result = $query->fields('n', array('nid'))
      ->orderBy('n.nid', 'ASC')
      ->condition('type', $node_types_array, 'IN')
      ->range($offset, $limit)->execute();
{% endhighlight %}

<p>
  Poi aggiorneremo l'URL utilizzando la funzione <code>pathauto_node_update_alias()</code>
  che troviamo all'interno del modulo <strong>Pathauto</strong>.
</p>
{% highlight php %}
  // Load pathauto.module from the pathauto module.
  module_load_include('module', 'pathauto', 'pathauto');

  $current_node = 0;
  foreach ($result as $row) {
    // Create the nids array
    $node = node_load($row->nid);
    pathauto_node_update_alias($node, 'update');

    // Update our progress information.
    $current_node = $row->nid;
  }
{% endhighlight %}

<h2 class="section-heading">Eseguire processi concorrenti</h2>
<p>
  Quello che dobbiamo fare ora è riuscire ad eseguire contemporaneamente la
  funzione descritta qui sopra su porzioni di dati differenti. Per farlo
  abbiamo bisogno di definire le seguenti cinque funzioni:
</p>
<ul>
  <li>
    <code>drush_mt_command()</code>, implementa il comando <code>mt-command</code>.
    Non fa altro che avviare la gestione multithread per l'aggiornamento di
    tutti i nodi;
  </li>
  <li>
    <code>_mt_command_setup()</code>, costruisce il comando Drush da eseguire
    per ogni singolo thread. Nel nostro caso il comando sarà <code>update-alias</code>;
  </li>
  <li>
    <code>_mt_command_teardown()</code>, se abbiamo bisogno di eseguire alcuni
    comandi al termine dell'esecuzione di un thread, li possiamo inserire qui;
  </li>
  <li>
    <code>_mt_monitor_process()</code>, chiude in modo sicuro il processo
    quando ha terminato la sua esecuzione;
  </li>
  <li>
    <code>drush_thread_manager()</code>, gestisce l'esecuzione di tutti i
    processi fino a completare l'aggiornamento del path alias per tutti i nodi.
  </li>
</ul>
<p>
  Nel mio caso le funzioni che ho personalizzato sono le prime due e le vedremo
  in dettagli qui di seguito. Mentre le altre le ho praticamente copiate
  e lasciate invariate come potete trovare nel
  <a href="https://github.com/johnennewdeeson/drush-multi-processing" title="Concurrent Applications with Drush" target="_blank">repository di GitHub</a>. La funzione <code>_mt_command_teardown()</code>
  l'ho lasciata vuota visto che non c'è bisogno di utilizzarla.
</p>

<h3>Setup di ogni singolo job</h3>
<p>
  Il compito di ogni singolo job è quello di aggiornare il path alias
  di un'insieme definito di nodi. Grazie ai parametri <code>$limit</code> e
  <code>$offset</code> posso identificare la prozione di nodi su cui lavorare.
  Il paramentro <code>$thread_id</code> l'ho aggiunto solo per poterlo stampare
  come log. La funzione <code>_mt_command_setup</code> alla fine restiuisce
  una stringa contentente la riga di comando drush che verrà eseguita dal
  gestore dei processi.
</p>
{% highlight php %}
function _mt_command_setup($thread_id, $limit, $offset) {
  $node_types = $GLOBALS['node_types'];
  $cmd = "drush update-alias $thread_id $limit $offset $node_types";
  return $cmd;
}
{% endhighlight %}

<h3>Il gestore dei processi concorrenti</h3>
<p>
  Chi è che inizializza e avvia il gestore di processi? Allora, ricollegandomi
  a quanto stavo dicendo qualche riga fa, questo compito è svolto dalla funzione
  <code>drush_mt_command()</code>. Per eseguire la funzione lanciamo il comando
  <code>$ drush mt-command [limit] [batch_size] [threads]</code> dove:
</p>
<ul>
  <li>
    <code>limit</code>, è il numero totale dei nodi che vogliamo aggiornare;
  </li>
  <li>
    <code>batch_size</code>, la quantità di nodi da elaborare per ogni job;</li>
  <li>
    <code>threads</code>, il numero di processi che possono essere in esecuzione
    contemporaneamente.
  </li>
</ul>
<p>
  Se vogliamo elaborare tutti i nodi presenti nel nostro database (nel mio caso
  ho deciso di aggiornare solo i contenuti di tipo 'article' e 'page'),
</p>
{% highlight php %}
  // Choose the content-type you will update.
  $node_types = array(
    'page',
    'article',
  );
{% endhighlight %}
<p>
  allora dobbiamo impostare a 0 il paramentro <code>limit</code>.
</p>
{% highlight php %}
  if($limit == 0) {
    // Retrieve all records
    $query = db_select('node', 'n');
    $result = $query->fields('n', array('nid'))
      ->orderBy('n.nid', 'ASC')
      ->condition('type', $node_types, 'IN')
      ->execute();

    $limit = $result->rowCount();
  }
{% endhighlight %}

<p>
  A questo punto è tutto pronto per avviare il gestore dei processi.
</p>
{% highlight php %}
  drush_thread_manager($limit, $batch_size, $threads, '_mt_command_setup',
    '_mt_command_teardown', $starting_offset);
{% endhighlight %}

<h3>Conclusioni</h3>
<p>
  Se dovete eseguire applicare qualche modifica a una grossa mole di dati
  del vostrp sito web in <em>Drupal</em>, penso che questo instroduzione alla
  <strong>programmazione multi-thread</strong> utilizzando <em>Drush</em> faccia al caso vostro.
</p>
<p>
  Nel <a href="https://github.com/robertoperuzzo/drush-multithread-update-alias" target="_blank" title="Drush Multi-Thread path alias update">mio repository GitHub</a> potete trovare il codice completo.
  Sono certo che si può far ancora meglio per velocizzare la procedura che ho
  descritto qui sopra, quindi se avete suggerimenti da darmi <a href="{{ '/contattami' | prepend: site.baseurl }}">contattatemi</a>, sarò felice di discuterne assieme.
</p>
