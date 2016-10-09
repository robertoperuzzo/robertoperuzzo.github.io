---
layout:   post
title:    "Aggiornare in massa l'urlalias dei contenuti"
subtitle: "Come utilizzare hook_update_N() e le Batch API."
date:     2016-10-07 13:42:00
tags:     drupal7 hook_update_N pathauto urlalias
---
<p>Quest'oggi volevo condividere con voi un
  <a href="https://www.drupal.org/node/2444939" target="_blank" title="Bulk updating Pathauto aliases - Using hook_update_N() and Batch API">post che avevo scritto</a> l'anno scorso
  su Drupal.org, utile per chi deve eseguire dei processi batch per aggiornare
  grosse moli di dati. Per evitare che il processo vada in timeout, invece di
  eseguire l'aggiornamento su tutto in un unica passata, effettueremo più
  aggiornamenti su porzioni di dati fino a completamento. Quando si deve lavorare
  con molti dati la soluzione sta sempre nel concetto di <em>divide et impera</em>.
</p>
<p>Qui di seguito vedremo i frammenti più significativi dell'intero codice che
potete trovare nel <a href="https://www.drupal.org/node/2444939" target="_blank" title="Bulk updating Pathauto aliases - Using hook_update_N() and Batch API">mio post</a>.</p>

<h2 class="section-heading">Utilizzare la variabile $sandbox</h2>
<p>
  Per partizionare l'insieme di dati da aggiornare dobbiamo tenere traccia di
  alcune informazioni importanti:
</p>
<ul>
  <li>il numero totale di nodi da aggiornare;</li>
  <li>il numero di nodi elaborati fino ad ora;</li>
  <li>messaggi da visualizzare durante l'esecuzione;</li>
  <li>l'identificativo dell'ultimo nodo aggiornato;</li>
</ul>
<p>Questi dati vengono memorizzati all'interno della variabile <code>$sandbox</code>
  passata come parametro (
  <a href="http://php.net/manual/en/language.references.pass.php" target="_blank" title="Passing by Reference">
    per riferimento</a>) alla funzione <code>hook_update_N(&$sandbox)</code>.
  Alla prima iterazione si inizializza la variabile con le informazioni di
  partenza nel modo seguente:</p>

{% highlight php %}
    // Use the sandbox to store the information needed to track progression.
    if (!isset($sandbox['progress'])) {
      // The count of nodes visited so far.
      $sandbox['progress'] = 0;
      // Total nodes that must be visited.
      $sandbox['max'] = $result->rowCount();
      // A place to store messages during the run.
      $sandbox['messages'] = array();
      // Last node read via the query.
      $sandbox['current_node'] = -1;
    } {% endhighlight %}

<p>Poi ad ogni iterazione successiva, provvederemo ad aggiornare i suoi valori:</p>

{% highlight php %}
  foreach ($result as $row) {
    /*
      Do you jobs...
     */

    // Update our progress information.
    $sandbox['progress']++;
    $sandbox['current_node'] = $row->nid;
  } {% endhighlight %}

<p>Prima di chiudere la funzione, verificheremo se il processo deve terminare
   o essere eseguito nuovamente. Per farlo basta confrontare il valore di
  <code>$sandbox['progress']</code> con il numero totale degli elementi da
  aggiornare <code>$sandbox['max']</code> nel seguente modo:</p>

{% highlight php %}
  // Set the "finished" status, to tell batch engine whether this function
  // needs to run again. If you set a float, this will indicate the progress
  // of the batch so the progress bar will update.
  $sandbox['#finished'] = ($sandbox['progress'] >= $sandbox['max'])
    ? TRUE
    : ($sandbox['progress'] / $sandbox['max']);

  if ($sandbox['#finished'])
  {
    return t('The batch URL Alias rebuild is finished.');
  } {% endhighlight %}

<h2  class="section-heading">Definire la finestra di elaborazione</h2>
<p>
  Dimenticavo, dovete inoltre impostare l'ampiezza della <em>finestra di elaborazione</em>,
  cioè il numero di elementi da elaborare ad ogni ciclo. Ecco come:</p>

{% highlight php %}
  // Process nodes by groups of 50.
  // When a group is processed, the batch update engine determines
  // whether it should continue processing in the same request or provide
  // progress feedback to the user and wait for the next request.
  $limit = 50;

  // Retrieve the next group of nids.
  $query = db_select('node', 'n')->extend('PagerDefault');

  $result = $query->fields('n', array('nid'))
      ->orderBy('n.nid', 'ASC')
      ->condition('type', array('type_one','type_two','type_three'), 'IN')
      ->condition('n.nid', $sandbox['current_node'], '>')
      ->limit($limit)->execute(); {% endhighlight %}

<p>
  Bene questo è tutto. Per vedere il codice completo vi rimando alla
  <a href="https://www.drupal.org/node/2444939" target="_blank" title="Bulk updating Pathauto aliases - Using hook_update_N() and Batch API">
    pagina del post</a> su Drupal.org.
</p>
