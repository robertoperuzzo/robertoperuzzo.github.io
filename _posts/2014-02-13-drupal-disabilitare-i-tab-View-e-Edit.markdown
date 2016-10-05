---
layout:     post
title:      "Drupal: disabilitare i tab Anteprima e Modifica"
subtitle:   "Impostare callback personalizzate tramite hook_menu_alter."
date:       2014-02-13 17:00:00
author:     "Roberto Peruzzo"
tags:       drupal hook_menu_alter access drupalcommerce
---

<p>Nel progetto che stiamo realizzando con Drupal Commerce mi si è posto
  difronte una problematica alquanto insolita: disabilitare il tab menu in
  alto a destra "View | Edit" in base al ruolo dell'utente loggato.</p>

<img src="{{ site.baseurl }}/img/2014-02-13/menu.png" alt="Menu image" />

<p>Dopo una prima ricerca in rete, ho visto che esiste il modulo
  <a href="https://www.drupal.org/project/tabtamer" target="_blank">Tab Tamer</a>
  che permette di manipolare questo menu, ma non da la possibilità di scegliere
  per quali ruoli abilitare o disabilitare le voci di menù. A questo punto ho
  sbirciato il codice del modulo e mi sono creato un semplicissimo modulo
  ad-hoc di cui pubblico il frammento di codice qui sotto.</p>

<pre><code>
  /**
   * Override hook_menu_alter().
   * Disable tab menu items by user role.
   *
   * @param  array $items [description]
   */
  function mymodule_menu_alter(&$items) {
    $items['node/%node/view']['access callback'] = '_mymodule_menu_view_access';
    $items['node/%node/edit']['access callback'] = '_mymodule_menu_edit_access';
  }

  /**
   * Check if user has access to 'view' tab item menu.
   *
   * @return boolean FALSE if current user has 'translator' role,
   *                 TRUE otherwise.
   */
  function _mymodule_menu_view_access() {
    global $user;

    if (in_array('web editor', $user->roles)
      || in_array('translator', $user->roles)) {
      return FALSE;
    }
    else {
      return TRUE;
    }
  }

  /**
   * Check if user has access to 'edit' tab item menu.
   *
   * @return boolean FALSE if current user has 'translator' role,
   *                 TRUE otherwise.
   */
  function _mymodule_menu_edit_access() {
    global $user;

    if (in_array('translator', $user->roles)) {
      return FALSE;
    }
    else {
      return TRUE;
    }
  }
</code></pre>
