---
layout:     post
title:      "Template diversi in base al URL alias per D7"
subtitle:   "Cosa è cambiato da Drupal 6 a Drupal 7."
date:       2012-04-11 12:53:00
author:     "Roberto Peruzzo"
tags:       drupal template urlalias theme_hook_suggestions
---

<p>Questa mattina ho scoperto che Drupal 7 ha modificato il modo di gestire il
  nome dei template files. Tempo fa, in un progetto con Drupal 6, avevo
  modificato la funzione <code>mytheme_preprocess_page(&$vars)</code> presente
  nel file <code>template.php</code> per poter gestire template diversi su
  pagine differenti basandomi sul URL alias della pagina stessa.</p>

<h2 class="section-heading">Il codice per D6</h2>
<pre><code>
  // Different page templates depending on URL aliases
  if (module_exists('path')) {
      $alias = drupal_get_path_alias(str_replace('/edit','',$_GET['q']));
      if ($alias != $_GET['q']) {
        $template_filename = 'page';
        foreach (explode('/', $alias) as $path_part) {
          $template_filename = $template_filename . '-' . $path_part;
          $vars['template_files'][] = $template_filename;
        }
      }
  }
</code></pre>

<p>In Drupal 7 questo codice non funziona perché è stata modificata la variabile
  che gestisce il nome del template. Non è più <code>$vars[‘template_files’]</code>,
  ma bensì <code>$vars['theme_hook_suggestions’]</code> e al posto del separatore
  <code>-</code>, viene utilizzato il doppio <em>underscore</em> <code>__</code>.</p>

<h2 class="section-heading">Il codice per D7</h2>
<pre><code>
  // Different page templates depending on URL aliases
  if (module_exists('path')) {
    $alias = drupal_get_path_alias(str_replace('/edit','',$_GET['q']));
    if ($alias != $_GET['q']) {
      $template_filename = 'page';
      foreach (explode('/', $alias) as $path_part) {
        $template_filename = $template_filename . '__' . $path_part;
        $vars['theme_hook_suggestions'][] = $template_filename;
      }
    }
  }
</code></pre>
<br />
<p>Dimenticavo il nome del template file ora dev'essere
  <code>page–<path_alias_part>.tpl.php</code>.</p>
