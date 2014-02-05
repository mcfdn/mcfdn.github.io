---
layout: post
status: publish
published: true
title: Checking for duplicate keys in Doctrine
author: James
author_login: James
author_email: james@jamesmcfadden.co.uk
wordpress_id: 426
wordpress_url: http://jamesmcfadden.co.uk/?p=426
date: 2013-01-30 10:49:30.000000000 +00:00
categories:
- PHP
- Doctrine
tags: []
---
Doctrine doesn't seem to have an inbuilt mechanism for checking for duplicate entries; one solution thrown around the web is to simply try catch the PDOException code 23000.

Here is a solution I use, which makes use of the MetadataFactory to retrieve the unique constraints. It then simply tries to find an existing entity based on the same values:

    public function isDuplicateEntity(Model $model) 
    {   
        $em = $this->getEntityManager;
        $cmf = $em->getMetadataFactory();
        $class = $cmf->getMetadataFor(get_class($model));
      
        if(isset($class->table['uniqueConstraints'])) {
            $uniqueColumns = array();
            $uniqueConstraints = $class->table['uniqueConstraints'];    
            
            foreach($uniqueConstraints as $constraintName => $constraintSection) {
                    foreach($constraintSection['columns'] as $column) {
                    $value = $model->{$column};
                        $uniqueColumns[$column] = $value;
                    }
            }
            $className = get_class($model);
            $unique = $em->getRepository($className)->findOneBy($uniqueColumns);
            
            if($unique instanceof $className) {
                $model->id = $unique->id;
                return true;
            }
        }
        return false;
    }
