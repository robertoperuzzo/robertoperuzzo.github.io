---
layout:     post
title:      "Paginated Scan on DynamoDB"
subtitle:   "Il nuovo servizio NoSQL Database offerto da Amazon."
date:       2012-04-11 12:43:00
author:     "Roberto Peruzzo"
tags:       java aws dynamodb amazon
---

<p>Riprendendo il <a href="{{ site.baseurl }}/2012/02/13/amazon-aws-dynamodb/">post</a>
  fatto un mese fa circa a proposito di DynamoDB ho potuto
  verificare che <strong>aumentando la capacità di scrittura e di lettura
  le prestazioni migliorano</strong>. Tanto per darvi un'idea, salvare un totale
  di 650 messaggi di dimensione random da 50byte a 1200byte ho misurato un
  <strong>tempo di latenza medio di circa 315ms</strong>. La latenza in questo
  caso corrisponde al tempo che trascorre dal momento in cui un client invia il
  messaggio al momento in cui riceve dal server la conferma che il messaggio è
  stato salvato nella tabella di DynamoDB.</p>

<p>Altra cosa da tener presente quando si effettua una scansione della tabella,
  il metodo <conde>scan()</code> è paginato. Questo significa che
  <strong>restituisce al massimo 1MB di dati alla volta</strong>. Se ci sono
  ancora dati, la funzione restituisce il valore <code>LastEvaluatedKey</code>
  che passandolo come parametro al metodo <code>scan()</code>, restituisce la
  "pagina" successiva di items.</p>
