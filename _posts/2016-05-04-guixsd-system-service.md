---
layout: default
title: Add Shepherd service to GuixSD
---
This post must be same tutorial for GuixSD as [https://nixos.org/wiki/NixOS:extend_NixOS](https://nixos.org/wiki/NixOS:extend_NixOS) for NixOS.

Lets create own guile modules structure:

    $ mkdir local-guix
    $ cd local-guix
    $ mkdir -p mnd/services # we will use service modules like (use-modules (mnd services service))
    $ edit irssi.scm # create irssi service

And create service that will run `irssi` under `screen`

Basic service
-------------

Firstly define module that will export our `irssi-service`

    (define-module (mnd services irssi)
      #:use-module (gnu services)
      #:use-module (gnu services shepherd)
      #:use-module (gnu packages irc) ; irssi
      #:use-module (gnu packages screen)
      #:use-module (guix gexp)
      #:use-module (guix records)
      #:use-module (ice-9 match)
      #:export (irssi-service))

Then define config for our module

    (define-record-type* <irssi-configuration>
      irssi-configuration make-irssi-configuration
      irssi-configuation?
      (irssi         irssi-configuration-irssi        ;<package>
                     (default irssi))
      (screen        irssi-configuration-screen       ;<package>
                     (default screen))
      (user          lirc-configuration-user)         ;string
      (group         lirc-configuration-group         ;string
                     (default "users"))
      (extra-options lirc-configuration-options       ;list of strings
                     (default '())))

Basicaly services extend `activation-service-type` and `shepherd-root-service-type`, but our service does not require to create runtime files so we'll extend only `shepherd-root-service-type`:

    (define irssi-shepherd-service
      (match-lambda
        (($ <irssi-configuration> irssi screen user group options)
            (list (shepherd-service
                   (provision '(irssi))
                   (documentation "Run the irssi client in screen.")
                   (requirement '(user-processes))
                   (start #~((make-forkexec-constructor
                              (list 
                               (string-append #$screen "/bin/screen")
                               "-D" "-m" "-S" "irc"
                               (string-append #$irssi "/bin/irssi")
                               #$@options))))
                   (stop #~(make-kill-destructor)))))))

For now we ignore  user and group options and just call `irssi` inside `screen`. This service will run irssi from root user.

Now we can define `irssi-service-type` that provides service with name `irssi` and extend `shepherd-root-service-type` with defined code.

    (define irssi-service-type
      (service-type (name 'irssi)
                    (extensions
                     (list (service-extension shepherd-root-service-type
                                              irssi-shepherd-service)))))

And finally define function that will create our `<service>` object:

    (define* (irssi-service #:key (irssi irssi)
                            (screen screen)
                            user
                            (group "users")
                            (extra-options '()))
      "irssi service for specific user" 
      (service irssi-service-type
               (irssi-configuration
                (irssi irssi) (screen screen)
                (user user) (group group)
                (extra-options extra-options))))

Run this service
----------------

To add service to you system you must modify you system configuration file in next manner.

1. Add `(use-modules (mnd services irssi))` to beginning of the file
2. Add `(irssi-service #:user "alice")` to list of services

And now you can add you new service to system with 

    $ su # log in as root
    $ guix system -L /path/to/local-guix reconfigure /path/to/your-system-config.scm
    ...
    making '/gnu/store/3j7wfbyhfs4pzrgdq6p8fnj4msyv01zc-system' the current system...
    guix system: loading new services: irssi...
    guix system: shepherd: Evaluating user expression (register-services (primitive-load "/gnu/s...")).
    guix system: shepherd: Service irssi has been started.
    
    Installation finished. No error reported.
    $ herd status irssi
    Status of irssi:
      It is started.
    ...

Add support for specific user
-----------------------------

You can look at code that support specific user at [github](https://github.com/mnd/guix-mnd-pkgs/blob/master/mnd/services/irssi.scm)

Debugging
---------

`Guix` and `shepherd` not very specific with errors. If something goes wrong you can try to load your module with `guile`:

    $ guile -L /path/to/local-guix -L ${HOME}/.config/guix/latest/
    scheme@(guile-user)> (use-modules (mnd services irssi))

**TODO** Some more usefull debug suggestions?
