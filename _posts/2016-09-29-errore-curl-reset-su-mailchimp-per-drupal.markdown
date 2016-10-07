---
layout:     post
title:      "Errore curl_reset su Mailchimp per Drupal"
subtitle:   "Lavorare con ambienti 'spaiati' ti fa saltar la cena."
date:       2016-09-29 12:00:00
author:     "Roberto Peruzzo"
header-img: "img/2016-09-29/hero-mailchimp.png"
tags:       drupal7 mailchimp development
---

<p>Quando l'ambiente dove sviluppi non ha le stesse cofingurazioni di quello
  di produzione, sta sicuro l'inghippo sta dietro l'angolo. L'ideale sarebbe di
  poter sempre lavorare con strumenti e piattaforme che ti permettono
  di avere ambienti allineati in modo da minimizzare al massimo eventuali
  problemi in fase di rilascio. Ma non sempre è così, soprattutto quando,
  come in questo caso, il cliente ha già il suo hosting o i suoi server dedicati.</p>

<p>Ieri, dopo aver caricato tutto il lavoro nell'ambiente di produzione, ecco apparire
  quel fastidioso messaggio <strong>Fatal error</strong>. Bene... si parte
  con il debugging motivato dalla solita imprecazione di circostanza.</p>

<p>Il messaggio che appare caricando l'homepage non l'ho mai visto prima<br>
  <pre><code>Fatal error: Call to undefined function GuzzleHttp\Handler\curl_reset() in /home/powell/public_html/sites/all/libraries/mailchimp/vendor/guzzlehttp/guzzle/src/Handler/CurlFactory.php on line 78</code></pre><br>
  e la cosa non mi piace, perché è sinonimo di "da questa sedia non ti alzerai
  per le prossime due ore" (tradotto significa "cena fredda e disappunto della
  mia cara moglie"). Per fortuna, a seguito di una veloce ricerca, ho trovato
  <a href="https://www.drupal.org/node/2709615#comment-11129049" target="_blank">questa
    segnalazione nel progetto Mailchimp</a> che mi ha dato il <em>LA</em> per
    risolvere il problema.</p>

<h2 class="section-heading">La soluzione</h2>

<p>All'interno della classe <code>CurlFactory.php</code> vien utilizzata la funzione <code>curl_reset()</code>
  disponibile nella versione <em>PHP >= 5.5.0</em>.<br> Se nel mio ambiente
  di sviluppo sto utilizzando <em>PHP 5.6.10</em>, nell'ambiente di produzione del
  cliente c'è <em>PHP 5.4.45</em>. La prima cosa che mi è venuto in mente di fare è
  aggiornare la versione di PHP, ma per evitare di allungare i tempi di consegna
  ho optato per <strong>disinstallare il modulo di Mailchimp 7.x-4.5 e sostituirlo
  con la versione 7.x-3.6</strong>. La versione precedente infatti utilizza la
  <a href="https://bitbucket.org/mailchimp/mailchimp-api-php" target="_blank">
  libreria 2.0.x delle API Mailchimp</a> che funziona con <em>PHP >= 5.2</em>.</p>
<p>In ogni caso evitate di lavorare su ambienti non allienali, eviterete di
ritardare la consegna e trovare la cena fredda.</p>
