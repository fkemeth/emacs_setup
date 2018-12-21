Emacs setup under Ubuntu 18.04
======

Install Emacs (26.1)
------

```
$ sudo apt-get install emacs26
```

Edit `~/.emacs`:

``` cl
;;; Configure
(setq name    USERNAME)     ;;; Max Mustermann
(setq email   EMAIL)        ;;; max@gmail.com
(setq server  SMTP_SERVER)  ;;; smtp.gmail.com
```

Create Credentials
------

Create credentials `~/.authinfo.gpg`, like
```
machine smtp.gmail.com login john_doe@gmail.com password "*secret*"
```
Make it accessible only to you
```
$ chmod 600 ~/.authinfo.gpg
```



Packages
-------

### mu4e (1.1.0)

 * Install mu via git
 ``` shell
 $ sudo apt-get install libgmime-3.0-dev libxapian-dev
 
 $ cd ~/Repos
 
 $ git clone git://github.com/djcb/mu.git
 
 $ cd mu
 $ ./autogen.sh && ./configure && make
 $ sudo make install
 $ cd ~/
 ```
 
 * Create maildir
  1. Install offlineimap:
  ``` shell
  $ sudo apt-get install offlineimap
  ```

  2. Configure offlineimap by creating `~/.offlineimap.py`:
  ``` python
  #! /usr/bin/env python3
  from subprocess import check_output
  
  def get_pass():
      string = check_output("gpg --use-agent --quiet --batch -d ~/.authinfo.gpg", shell=True).decode("utf-8")
      string = string.split(' ')
      passwd = string[string.index('password')+1]
      passwd = passwd.replace('"', '')
      return passwd
  
  def get_user():
      string = check_output("gpg --use-agent --quiet --batch -d ~/.authinfo.gpg", shell=True).decode("utf-8")
      string = string.split(' ')
      user = string[string.index('login')+1]
      user = user.replace('"', '')
      return user
  
  def get_host():
      string = check_output("gpg --use-agent --quiet --batch -d ~/.authinfo.gpg", shell=True).decode("utf-8")
      string = string.split(' ')
      host = string[string.index('machine')+1]
      host = host.replace('smtp', 'imap')
      return host  
  ```
  and editing `~/.offlineimaprc`
  ```
  [general]
  accounts = Goneo
  maxsyncaccounts = 3
  pythonfile = ~/.offlineimap.py
  
  [Account Goneo]
  localrepository = Local
  remoterepository = Remote
  
  [Repository Local]
  type = Maildir
  localfolders = ~/Mail
  
  [Repository Remote]
  type = IMAP
  remotehosteval = get_host()
  remoteusereval = get_user()
  remotepasseval = get_pass()
  ssl = yes
  maxconnections = 1
  sslcacertfile = /etc/ssl/certs/ca-certificates.crt
  ```
  
  3. Create Maildir
  ``` shell
  $ offlineimap
  ```
  4. Check if everything works
  ``` shell
  $ mu index --maildir=~/Mail
  ```
   
 * Configure `~/.emacs` for  mu4e

 ``` cl
;;; Configure mu4e
(add-to-list 'load-path "~/Repos/mu/mu4e/")

(require 'mu4e)

;; use mu4e for e-mail in emacs
(setq mail-user-agent 'mu4e-user-agent)
(setq mu4e-mu-binary "~/Repos/mu/mu/mu")

;; default
;; these are actually the defaults
(setq
  mu4e-maildir       "~/Mail"   ;; top-level Maildir
  mu4e-sent-folder   "/Sent"       ;; folder for sent messages
  mu4e-drafts-folder "/Drafts"     ;; unfinished messages
  mu4e-trash-folder  "/Trash"      ;; trashed messages
  mu4e-refile-folder "/Archives")   ;; saved messages 

;; setup some handy shortcuts
;; you can quickly switch to your Inbox -- press ``ji''
;; then, when you want archive some messages, move them to
;; the 'All Mail' folder by pressing ``ma''.

(setq mu4e-maildir-shortcuts
    '( ("/INBOX"               . ?i)
       ("/Sent"                . ?s)
       ("/Trash"               . ?t)
       ("/All Mail"            . ?a)))

;; allow for updating mail using 'U' in the main view:
(setq mu4e-get-mail-command "offlineimap")

;; something about ourselves
;; something about ourselves
(setq
   user-mail-address email
   user-full-name name
    )

;; sending mail -- replace USERNAME with your gmail username
;; also, make sure the gnutls command line utils are installed
;; package 'gnutls-bin' in Debian/Ubuntu

(require 'smtpmail)
(setq message-send-mail-function 'smtpmail-send-it
   starttls-use-gnutls t
   smtpmail-auth-credentials "~/.authinfo.gpg"
   smtpmail-default-smtp-server server
   smtpmail-smtp-server server
   smtpmail-smtp-service 587)

(setq message-kill-buffer-on-exit t)

;; use 'fancy' non-ascii characters in various places in mu4e
(setq mu4e-use-fancy-chars t)

;; save attachment to my desktop (this can also be a function)
(setq mu4e-attachment-dir "~/Desktop")

;; attempt to show images when viewing messages
(setq mu4e-view-show-images t)

(setq
  mu4e-get-mail-command "offlineimap"  
  mu4e-update-interval 300)             ;; update every 5 minutes

 ```