---
layout:     post
title:      "Il deploy delle immagini di default"
subtitle:   "Come impostare automaticamente la default image per i campi di tipo immagine."
date:       2016-10-06 22:25:00
author:     "Roberto Peruzzo"
tags:       drupal7 deployment hook_update_N default_image_ft
---

<p>Il problema di fondo è che i file delle immagini di default che imposti nel tuo
  ambiente di sviluppo non possono essere trasferite in modo automatico
  nei vari ambienti con il solo utilizzo di
  <a href="https://www.drupal.org/project/features" target="_blank" title="Features module">
    Features</a>.
  Quando imposti un'immagine di default e poi esporti con Features quel campo,
  ciò che viene memorizzato è il <code>fid</code> del file che hai caricato.
  Ma quel file è presente nel tuo ambiente e non negli altri; inoltre non puoi
  garantire che l'identificativo fid del tuo file possa essere lo stesso dappertutto
  perché i database dei vari ambienti hanno una vita propria.</p>

<h2 class="section-heading">La soluzione</h2>

<p>Per risolvere la questione bisogna:</p>
<ol>
  <li>mappare il fid del file immagine da usare tramite una variabile Drupal;</li>
  <li>caricare il file in modo automatico in ogni ambiente.</li>
</ol>

<p>Il primo punto lo risolviamo utilizando il modulo
  <a href="https://www.drupal.org/project/default_image_ft" target="_blank" title="Default image ft module">
    Default image ft</a>. In questo modo possiamo scegliere il nome della variabile che
  immagazzinerà il fid dell'immagine (vedi ad esempio l'immagine seguente).
  <img src="{{ site.baseurl }}/img/2016-10-06/source_variable.png" alt="Source image variable name" />
  <span class="caption text-muted">Impostazione che trovi nella pagina di
    modifica del campo immagine (es. admin/structure/types/manage/CONTENT_TYPE/fields/field_MY_IMAGE</span></p>

<p>A questo punto possiamo <a href="https://www.drupal.org/docs/7/creating-custom-modules" target="_blank" title="Creating custom modules">
  creare il nostro modulo custom</a> per caricare l'immagine
  in modo automatico. Le cose da fare sono due: posizionare l'immagine in una
  sotto-cartella del modulo, ad esempio <code>MY_MODULE_NAME/images/my_default_image.jpg</code>
  poi aggiungere il
  <a href="https://www.drupal.org/docs/7/creating-custom-modules/writing-install-files-drupal-7x" target="_blank">
  file di installazione</a> MY_MODULE_NAME.install. Inseriremo al suo interno la funzione
  <a href="https://api.drupal.org/api/drupal/modules%21system%21system.api.php/function/hook_update_N/7.x" target="_blank" title="function hook_update_N">
    <code>hook_update_N()</code>
  </a> per caricare fisicamente l'immagine dentro
  la directory <code>public://default_images</code> e per assegnarle un <code>fid</code>.
  Qui di seguito vediamo i passaggi più importanti.
</p>

<h3>Crea la cartella di destinazione se non esiste</h3>
<p>Ho scoperto l'esistenza di una funziona Drupal molto utile:
  <a href="https://api.drupal.org/api/drupal/includes!file.inc/function/file_prepare_directory/7.x" target="_blank" title="function file_prepare_directory">
    <code>file_prepare_directory()</code>
  </a>. Essa ci permette di verificare se
  la directory esiste; nel caso non esista e Drupal ha i permessi di scrittura
  nel filesystem, allora la crea.</p>

<h3>Salva il file e ottieni il fid</h3>
<p>Per prima cosa dobbiamo creare l'oggetto file Drupal. Per farlo dobbiamo
  creare un oggetto PHP di tipo <code>stdClass</code> e assegnarli gli attributi
  descritti
  <a href="https://api.drupal.org/api/drupal/includes!file.inc/group/file/7.x" target="_blank" title="File interface">
  in questa pagina</a> per l'immagine sorgente. Qui di seguito trovate il frammento di codice.
</p>
<pre>
  <code>
    // Create the surce file.
    $file = new stdClass;
    $file->filename = $filename;
    $file->timestamp = REQUEST_TIME;
    $file->uri = $source;
    $file->filemime = file_get_mimetype($source);
    $file->uid = 1;
    $file->status = 1;
  </code>
</pre>

<h3>Copia il file</h3>
<p>In Drupal esiste la funzione
  <a href="https://api.drupal.org/api/drupal/includes!file.inc/function/file_copy/7.x" target="_blank" title="function file_copy">
    <code>file_copy()</code>
  </a>
  che ti aiuta in questo compito. Dal frammento di codice seguente possiamo vedere
  che questa funzione restitusce l'oggetto del file copiato. Questo ci permette
  di ottenere il <code>fid</code> che gli è stato assegnato da Drupal e
  che salvereno nella nostra variabile.
</p>
<pre>
  <code>
    // Copy the file into destination directory.
    $file = file_copy($file, $destination_dir, FILE_EXISTS_REPLACE);
    // Get the fid.
    $fid = $file->fid;
  </code>
</pre>

<h3>Salva il fid nella variabile utilizzata dal modulo default_image_ft</h3>
<p>
  In conclusione utilizzeremo la funzione
  <a href="https://api.drupal.org/api/drupal/includes!bootstrap.inc/function/variable_set/7.x" target="_blank" title="function variable_set">
    <code>variable_set()</code>
  </a> per memorizzare il fid dell'immagine.
</p>
<pre>
  <code>
    // Set the default_image_ft variable.
    variable_set('my_default_image_variable', $fid);
  </code>
</pre>

<h3>Il codice completo da inserire nel file .install</h3>
<blockquote>
  Sostituisci i segnaposto:
  <ul>
    <li>MY_MODULE_NAME, con il nome del tuo modulo;</li>
    <li>my_default_image_variable, con il nome della tua variabile;</li>
    <li>my_default_image.jpg, con il nome della tua immagine.</li>
  </ul>
</blockquote>
<pre>
  <code>
    /**
     * Helper to replace default field base image.
     *
     * @param $filename
     * @param $source
     */
    function _MY_MODULE_NAME_replace_default_image($filename, $source) {
      // Set the destination dir.
      $destination_dir = 'public://default_images';

      // Create new file object and get new fid.
      if (file_exists($source)
        && file_prepare_directory($destination_dir, FILE_CREATE_DIRECTORY)) {

        // Create the surce file.
        $file = new stdClass;
        $file->filename = $filename;
        $file->timestamp = REQUEST_TIME;
        $file->uri = $source;
        $file->filemime = file_get_mimetype($source);
        $file->uid = 1;
        $file->status = 1;

        // Copy the file into destination directory.
        $file = file_copy($file, $destination_dir, FILE_EXISTS_REPLACE);
        $fid = $file->fid;

        // Set the default_image_ft variable.
        variable_set('my_default_image_variable', $fid);
        watchdog('MY_MODULE_NAME', 'Setted my_default_image_variable fid default image.');
      }
    }

    /**
     * Upload the default images for your image filed.
     */
    function MY_MODULE_NAME_update_7100() {
      $filename = 'my_default_image.jpg';
      $source = drupal_get_path('module', 'MY_MODULE_NAME')
        . '/images/' . $filename;
      // Replace default image for $field_name field base.
      _MY_MODULE_NAME_replace_default_image($filename, $source);
    }
  </code>
</pre>

