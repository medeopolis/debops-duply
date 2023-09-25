debops-duply
============

Wrap up oefenweb.duply_backup to configure Duply, a Duplicity wrapper, to ease integration with DebOps.

Requirements
------------

Any pre-requisites that may not be covered by Ansible itself or the role should be mentioned here. For instance, if the role uses the EC2 module, it may be a good idea to mention in this section that the boto package is required.

Extra packages will need to be installed depending on the Duplicity backends being used. Examples are:

* For OpenStack Swift, install `python3-swiftclient` and `python3-keystoneclient`.
* For scp install `openssh-client`
* For rsync over SSH install `openssh-client` and `rsync`
* For S3 compatible services install `python3-boto3` (use `boto3+s3:///` as the protocol)


Configuring `duply__apt_(group|host)_packages` will ensure they're installed where appropriate.


Known problems
--------------

* Requires two custom branches present in https://github.com/medeopolis/ansible-duply-backup (they are currently being upstreamed)
* Requires user to merge `duply__profile_base` with each profile provided in `duply__backup_combined_profiles` in the inventory. In future this should be moved to a role task.
* Variable naming is inconsistent and will need to be updated in a future release

Role Variables
--------------

Like other DebOps roles, `debops-duply` has several layers of variables which can be used for a given task. `defaults/main.yml` has the full details, this list just provides an overview.

* `duply__apt*_packages`: Lists of packages to be installed via debops.apt_install
* `duply__profile_conf*_gpg_encrypting_keys`: Lists of keys to use for encrypting backups
* `duply__profile_conf_environment_exports`: Lists of exports in to the environment ; can be used for various reasons but the main # one is configuring transports (targets)
* `duply__profile_excludes*`: Lists of files to exclude from **all** backups
* `duply__profile_conf*_duply_parameters`: Lists of extra parameters to provide the Duplicity command via Duply
* `duply__profile_conf_gpg_key`: Key to encrypt with
* `duply__profile_conf_gpg_key_password`: password to encrypting key (if set)
* `duply__profile_conf_gpg_signing_key`: Key to sign with, setting to duply__profile_conf_gpg_key is likely
* `duply__profile_conf_gpg_options`: Extra options for Duplicity to set for GPG 
* `duply__profile_conf_backup_target`: Required - backup destination
* `duply__profile_conf_backup_source`: What is being backed up
* `duply__profile_conf_max_full_backups`: How full backups will be retained
* `duply__profile_conf_max_full_backup_age`: How long between full backups
* `duply__profile_conf_max_fulls_with_incrementals`: How many full backups should have incremental backups retained
* `duply__profile_conf_backup_volume_size`: Size of backup volumes
* `duply__profile_conf_archive_directory`: Location for Duplicities cache.
* `duply__profile_conf_temporary_directory`: Where Duplicity temp files (including downloaded archives) are stored. This needs to be a filesystem with a substantial amount of space.
* `duply__profile_base`: Defaults for settings to pass to oefenweb.duply_backup. Other profiles should be merged with this in the inventory, or duplicate all its settings (as appropriate)
* `duply__gpg_*`: Settings related to GPG handling. Only required if encrypting with GPG
* `duply__cron*_jobs`: `debops.cron` formatted cronjobs; these are passed directly to debops.cron for processing.

See defaults/main.yml for the full list.


Dependencies
------------

- Requires oefenweb.duply_backup (which does all the Duply configuration)
- Requires DebOps to be in use, particularly apt_install and cron.
- Tested with medeopolis.debops-gpgkey but any other key generation would be fine if configured appropriately

Example Playbook
----------------

Playbook including two essential debops roles (core and PKI), plus optional GPG
key generation.
This example is for illustrative purposes only.

    - name: Configure Duply with backups to Openstack Swift storage with GPG keys configured by medeopolis.debops-gpgkey
      hosts: debops_service_duply
      become: true
      become_user: root
      vars:
        gpgkey__generate_home: "{{ pki_root }}/gpg/duply"
        gpgkey__generate_realname: "Generated for Duply backups"
        gpgkey__generate_email: "noreply+{{ inventory_hostname_short }}@example.com"
        gpgkey__generate_passphrase_file: 'duply-backups-gpg-passphrase.key'
        gpgkey__export_public_key_name:  "duply-backup-key.pub"
        gpgkey__export_private_key_name: "duply-backup-key.priv"
        gpgkey__export_fingerprint_name: "duply-backup-key.fingerprint"

        duply__profile_conf_gpg_signing_key: "{{ lookup('ansible.builtin.file', gpgkey__retrieve_target + '/' + gpgkey__export_fingerprint_name ) }}"
        duply__profile_conf_gpg_key_password: "{{ gpgkey__generate_passphrase }}"
				duply__profile_conf_combined_gpg_encrypting_keys: "{{ duply__profile_conf_gpg_signing_key }}"
        duply__profile_conf_gpg_options: "--pinentry-mode loopback --homedir {{ gpgkey__generate_home }}/.gnupg"

				duply__apt_group_packages:
					- python3-swiftclient
					- python3-keystoneclient

				duply__profile_conf_backup_target: 'swift://my_object_store'
				duply__profile_conf_backup_volume_size: 500
				duply__profile_conf_max_full_backups: 4
				duply__profile_conf_max_fulls_with_incrementals: 1
				duply__profile_conf_group_environment_exports:
							 - name: SWIFT_USERNAME
								 value: customer@example.com
							 - name: SWIFT_PASSWORD
								 value: secure-password-from-swift-vendor

				duply__profile_conf_archive_directory: '/bulk/duply_archive'
				duply__profile_conf_temporary_directory: "{{ duply__profile_conf_archive_directory + '/tmp' }}"


				os_excludes:
					- '- /srv/duplicity_archive_dir/'
					- '- /swapfile'
				duply_operating_system_profile:
							conf:
								# Default source is /
								dupl_params: "{{ duply__profile_conf_combined_duply_parameters + ' --exclude-other-filesystems ' }}"
								target: "{{ duply__profile_conf_backup_target + '/' + inventory_hostname + '/system' }}"
								export: "{{ duply__profile_conf_combined_environment_exports }}"
							excludes: "{{ duply__profile_excludes_combined + os_excludes }}"
				duply__backup_profiles:
					"operating_system": "{{ duply__profile_base |combine(duply_operating_system_profile, recursive=true) }}"
				duply__cron_jobs:
					"operating_system_backup":
						hour: '01'
						minute: '00'
						user: root
						job: "duply operating_system backup"
					"operating_system_cleanup":
						hour: '09'
						minute: '00'
						user: root
						weekday: 'sat'
						job: "duply operating_system cleanup --force ; duply operating_system purgeAuto --force"

      roles:
        - role: debops.debops.core
        - role: debops.debops.pki
        - role: medeopolis.debops-gpgkey
        - role: medeopolis.debops-duply



License
-------

GPLv3

