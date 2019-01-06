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

;;; Start in full screen
(custom-set-variables
 '(initial-frame-alist (quote ((fullscreen . maximized)))))
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

5. Enable mu4e alerts
``` shell
$ cd ~/Repos/emacs/
$ git clone https://github.com/jwiegley/alert
$ git clone https://github.com/magnars/s.el.git
$ git clone https://github.com/Wilfred/ht.el.git
$ git clone https://github.com/magnars/dash.el.git
$ git clone https://github.com/iqbalansari/mu4e-alert
$ git clone https://github.com/jrblevin/markdown-mode.git
$ git clone https://code.orgmode.org/bzg/org-mode.git
```
 
* Configure `~/.emacs` for  mu4e

``` cl
;;; Configure mu4e
;;; Configure mu4e
(add-to-list 'load-path "~/Repos/mu/mu4e/")

(require 'mu4e)

;; use mu4e for e-mail in emacs
(setq mail-user-agent 'mu4e-user-agent)
(setq mu4e-mu-binary "~/Repos/mu/mu/mu")

;; set headers
(setq mu4e-headers-fields
    '( (:date          .  25)    ;; alternatively, use :human-date
       (:flags         .   6)
       (:from          .  22)
       (:thread-subject .  nil))) ;; alternatively, use :thread-subject

;; set mail folders
(setq
  mu4e-maildir       "~/Mail"   ;; top-level Maildir
  mu4e-sent-folder   "/Sent"       ;; folder for sent messages
  mu4e-drafts-folder "/Drafts"     ;; unfinished messages
  mu4e-trash-folder  "/Trash"      ;; trashed messages
)

;; setup some handy shortcuts
;; you can quickly switch to your Inbox -- press ``ji''
(setq mu4e-maildir-shortcuts
    '( ("/INBOX"               . ?i)
       ("/Sent"                . ?s)
       ("/Trash"               . ?t)
       ("/Archives"            . ?a)
       ("/INBOX.Rechnungen"    . ?r)
       ("/INBOX.Meeetup"       . ?m)))

;; dynamice refiling
(setq mu4e-refile-folder
  (lambda (msg)
    (cond
      ;; messages to the mu mailing list go to the /mu folder
      ((mu4e-message-contact-field-matches msg :to
         "mu-discuss@googlegroups.com")
        "/mu")
      ;; messages sent directly to me go to /archive
      ;; also `mu4e-user-mail-address-p' can be used
      ;; ((mu4e-message-contact-field-matches msg :to "me@example.com")
      ;;   "/private")
      ;; messages with football or soccer in the subject go to /football
      ((string-match "Rechnung\\|Rechnungen"
         (mu4e-message-field msg :subject))
        "/INBOX.Rechnungen")
      ((string-match "Meetup\\|Meetups"
         (mu4e-message-field msg :subject))
        "/INBOX.Meetup")
      ;; messages sent by me go to the sent folder
      ((find-if
         (lambda (addr)
           (mu4e-message-contact-field-matches msg :from addr))
         mu4e-user-mail-address-list)
        mu4e-sent-folder)
      ;; everything else goes to /archive
      ;; important to have a catch-all at the end!
      (t  "/Archives"))))

;; allow for updating mail using 'U' in the main view:
(setq mu4e-get-mail-command "offlineimap")

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

;; save attachment to my desktop (this can also be a function)
(setq mu4e-attachment-dir "~/Desktop")

;; attempt to show images when viewing messages
(setq mu4e-view-show-images t)

(setq
  mu4e-get-mail-command "offlineimap"  
  mu4e-update-interval 300)             ;; update every 5 minutes

;; (require 'alert)
(add-to-list 'load-path "~/Repos/emacs/alert/")
(add-to-list 'load-path "~/Repos/emacs/s.el/")
(add-to-list 'load-path "~/Repos/emacs/dash.el/")
(add-to-list 'load-path "~/Repos/emacs/ht.el/")

(add-to-list 'load-path "~/Repos/emacs/mu4e-alert/")
(require 'mu4e-alert)
(mu4e-alert-set-default-style 'libnotify)
(add-hook 'after-init-hook #'mu4e-alert-enable-notifications)
 ```
 


### Monakai Color Theme

1. Load color theme

```
$ cd ~/Repos/emacs/
$ git clone https://github.com/oneKelvinSmith/monokai-emacs
```

2. Edit `~/.emacs`:

``` cl
(add-to-list 'custom-theme-load-path "~/Repos/emacs/monokai-emacs/")
(load-theme 'monokai t)

(setq monokai-height-minus-1 0.8
      monokai-height-plus-1 1.1
      monokai-height-plus-2 1.15
      monokai-height-plus-3 1.2
      monokai-height-plus-4 1.3)
```

### Syntax Highlighting

1. Markdown
Edit `~/.emacs`:

``` cl
;;; Markdown Mode
(add-to-list 'load-path "~/Repos/emacs/markdown-mode/")
(require 'markdown-mode)
```

Contacts
------
Transform mu contact list into org-contacts
``` shell
$ mkdir ~/Contacts
$ mu cfind --format=org-contact >> ~/Contacts/contacts.org
```
Edit `~/.emacs`:
``` cl
;;; Contacts
(add-to-list 'load-path "~/Repos/emacs/org-mode/contrib/lisp/")

(require 'org-contacts)
(setq org-contacts-files '("~/Contacts/contacts.org"))

(defun write-contact (str)
  "Write new contact entry to org contact file."
  (interactive)
  (setq contact-name (read-string "Enter name: "))
  (write-region (concat "* " contact-name "\n:PROPERTIES:\n:EMAIL: " str "\n:END:\n") nil (car org-contacts-files) 'append)
  )

(defun mu4e-check-if-new-address (str)
  "From https://www.reddit.com/r/emacs/comments/76s9dl/orgmode_for_contacts/:
Find STR in my address book file."
  (interactive)
  (let ((search-str (replace-regexp-in-string "+" " " str))
        (current-buf (current-buffer))
	)
    (find-file (car org-contacts-files))
    (goto-char (point-min))
    (or
     (search-forward search-str nil t)
     (if (y-or-n-p "Unknown email address. Add to contacts?")
	 (write-contact search-str)
       ()
       )))
  (switch-to-buffer (other-buffer (current-buffer) 1))
  )
(defun mu4e-headers-view-message ()
  "View message at point.
If there's an existing window for the view, re-use that one. If
not, create a new one, depending on the value of
`mu4e-split-view': if it's a symbol `horizontal' or `vertical',
split the window accordingly; if it is nil, replace the current
window. "
  ;;; This should not be a global address. Modify this!
  (setq mu4e-sender-address (cdr-safe (car-safe (mu4e-message-field (mu4e-message-at-point) :from))))
  (mu4e-check-if-new-address mu4e-sender-address)
  (interactive)
  (unless (eq major-mode 'mu4e-headers-mode)
    (mu4e-error "Must be in mu4e-headers-mode (%S)" major-mode))
  (let* ((msg (mu4e-message-at-point))
	  (docid (or (mu4e-message-field msg :docid)
		     (mu4e-warn "No message at point")))
	  (addr (mu4e-message-field msg :from))
	  (decrypt (mu4e~decrypt-p msg))
	  (viewwin (mu4e~headers-redraw-get-view-window)))
    (unless (window-live-p viewwin)
      (mu4e-error "Cannot get a message view"))
    (select-window viewwin)
(switch-to-buffer (mu4e~headers-get-loading-buf))
    (mu4e~proc-view docid mu4e-view-show-images decrypt))
    )

```
Getting Things Done
------
Edit `~/.emacs`

``` cl
;;; Configure org-mode

;;; Setting up gtd
(setq org-agenda-files '("~/gtd/inbox.org"
                         "~/gtd/gtd.org"
			 "~/gtd/someday.org"
                         "~/gtd/tickler.org"))

;;; org-capture
(define-key global-map "\C-cc" 'org-capture)

;;; capture templates
(setq org-capture-templates '(("t" "Todo [inbox]" entry
                               (file+headline "~/gtd/inbox.org" "Tasks")
                               "* TODO %i%?")
                              ("T" "Tickler" entry
                               (file+headline "~/gtd/tickler.org" "Tickler")
                               "* %i%? \n %U")))

;;; refile targets
(setq org-refile-targets '((nil :maxlevel . 5) (org-agenda-files :maxlevel . 5)))

;;; set time stemp when done
(setq org-log-done 'time)
;;; set tags
(setq org-tag-alist '(("work" . ?w) ("home" . ?h) ("read" . ?r)("organize" . ?o)))

;;; org-latex-export-to-pdf keybind
(eval-after-load "org"
  '(progn
     (define-key org-mode-map (kbd "\C-C \C-c") 'org-latex-export-to-pdf)
     )
  )

```

