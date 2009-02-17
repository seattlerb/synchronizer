# Seattle.rb Project Synchronizer

A simple, stupid tool for keeping GitHub mirrors of Seattle.rb
Perforce projects.

## Setup

Make sure you have Python, `vendor/git-p4` needs it. You'll also need
to be set up in [Zenspider's Perforce repo][p4].

[p4]: http://zenspider.com/ZSS/Process/Perforce.html

Tell Git about your Perforce config:

    $ git config --global git-p4.user YOUR_USERNAME
    $ git config --global git-p4.password YOUR_PASSWORD
    $ git config --global git-p4.port p4.zenspider.com:1666
    $ git config --global git-p4.client YOUR_CLIENT

## Synchronizing Existing Stuff

    $ rake sync

This will pull changes for everything listed in `projects.yml` into
repositories under `projects/` and push to GitHub. It'll handle new or
existing projects, but **make sure the seattlerb GitHub project repo
exists first**. You'll need to be a project collaborator to push.

GitHub project creation and collaborator management could probably be
automated, but I don't care.
