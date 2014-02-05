---
layout: post
status: publish
published: true
title: BlueImp jQuery File Upload in Zend Framework
author: James
author_login: James
author_email: james@jamesmcfadden.co.uk
wordpress_id: 276
wordpress_url: http://jamesmcfadden.co.uk/?p=276
date: 2012-10-23 17:18:50.000000000 +01:00
categories:
- PHP
- Zend Framework
tags: []
---
I came across the popular [jQuery File Upload](https://github.com/blueimp/jQuery-File-Upload) plugin while looking for a simple to use and extensible file upload solution to integrate into a Zend Framework project.

It can be a bit tricky to get this plugin working, so I thought I'd document the steps I took to get it working in ZF.

### Form Class

You could skip this step and HTML-ify your form by hand in your view, but that would defeat the point of this post, so this is what my form class looks like. **The element HTML classes are required**:

    /**
     * This is aimed for use with jQueryFileUpload
     * 
     * @link https://github.com/blueimp/jQuery-File-Upload
     * @author James McFadden <james@jamesmcfadden.co.uk>
     */
    class Form_Jquery_File_Upload extends Form_Upload
    {
        function init()
        {
            $this->resolveDependencies();
            
            $this->setAttrib('id', 'fileupload');
            
            $this->addElement(
                'file',
                'Photo'
            );
            $this->addElement(
                'Button',
                'Start',
                array(
                    'label' => 'Start Upload',
                    'class' => 'btn btn-primary start',
                    'escape' => false
                )
            );
            
            $this->addElement(
                'Button',
                'Cancel',
                array(
                    'label' => 'Cancel Upload',
                    'class' => 'btn btn-warning cancel',
                    'escape' => false
                )
            );
            $this->addElement(
                'Button',
                'Delete',
                array(
                    'label' => 'Delete',
                    'class' => 'btn btn-danger delete',
                    'escape' => false
                )
            );
        }
        
        private function resolveDependencies()
        {
            // Include your jQueryFileUpload scripts here
        }
    }

### Form Partial

Set up your form to render in a partial script (see [this useful post](http://devzone.zend.com/1240/decorators-with-zend_form) about Zend_Form decorators and enabling view scripts) like so:

    <form id=" echo $this->escape($this->form->getAttrib('id')); ?>" action=" echo $this->escape($this->form->getAction()); ?>" method=" echo $this->form->getMethod(); ?>">
        <div class="fileupload-buttonbar">
            <span class="fileinput-button">
            <span>Add files...</span>
                 echo $this->form->Photo; ?>
            </span>
             echo $this->form->Start; ?>
             echo $this->form->Cancel; ?>
             echo $this->form->Delete; ?>
            <div class="fileupload-progress fade">
                <div class="progress progress-success progress-striped active" role="progressbar" aria-valuemin="0" aria-valuemax="100">
                    <div class="bar" style="width:0%;"></div>
                </div>
                <div class="progress-extended">&nbsp;</div>
            </div>
        </div>
        <div class="fileupload-loading"></div>
        <br>
        <table role="presentation" class="table table-striped"><tbody class="files" data-toggle="modal-gallery" data-target="#modal-gallery"></tbody></table>
    </form>

    <!-- Here you should include the rest of the jQuery File Upload template -->

At this point you should create your form processing action, which should upload your photo to a specified directory on your server. I won't be going into file uploads here as it is slightly out of the scope of this post. As specified in the [jQuery File Upload documentation](https://github.com/blueimp/jQuery-File-Upload/wiki/Setup) you must return a JSON array containing information about the file uploaded. This _must_ be a JSON array, regardless of the number of files uploaded, as jQuery File Upload will use this information to present feedback to the user. A basic example might be:

    $jsonArray = array();
    
    foreach($photos as $i => $photo) {
        $jsonArray[$i]['name'] = $photo->getName();
        $jsonArray[$i]['size'] = $photo->getSize();
        $jsonArray[$i]['url'] = $photo->getUrl();
        $jsonArray[$i]['thumbnail_url'] = $photo->getThumbnailUrl();
        $jsonArray[$i]['delete_url'] = 'delete-photo.php?id=' . $photo->getId();
    }
    echo json_encode($jsonArray);

jQuery File Upload should do it's magic, and your files should be uploaded to your server, providing you have written your upload script correctly :)
