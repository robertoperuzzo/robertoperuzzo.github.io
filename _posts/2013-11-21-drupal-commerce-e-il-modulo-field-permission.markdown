---
layout:     post
title:      "Drupal Commerce e il modulo Field Permission"
subtitle:   "Il problema dei locked fields"
date:       2013-11-21 12:00:00
author:     "Roberto Peruzzo"
header-img: "img/2013-11-21/fields_permission.jpg"
tags:       drupal7 drupalcommerce field_permission
---

<p>Oggi ho scoperto che in <a href="https://www.drupal.org/project/commerce" target="_blank">Drupal Commerce</a>
  alcuni campi non possono essere editati a piacere (locked fields).
  Nel caso specifico sto parlando del campo prezzo. A quanto pare sembra
  essere un'imposizione dettata dalla struttura di Drupal Commerce motivata dal
  fatto che <strong>il prezzo viene ricalcolato</strong> per ogni nuovo utente/acquirente che si
  registra. Questo permette di definire prezzi diversi in base all'utente (fonte
  <a href="www.drupalcommerce.org" target="_blank">www.drupalcommerce.org</a>).</p>

<p>Questa caratteristica di Drupal Commerce non mi permette di utilizzare il
  modulo <a href="https://www.drupal.org/project/field_permissions" target="_blank">Field Permission</a>
  per disabilitare la modifica e l'inserimento del campo prezzo per alcuni ruoli utente.</p>
