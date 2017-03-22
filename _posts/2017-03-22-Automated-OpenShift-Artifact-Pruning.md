---
title: Automated Pruning of OpenShift Artifacts; Builds, Deploys, Images
layout: post
tags:
 - openshift
 - OCP3.4
---

After running openshift for a while I discovered that letting builds pile up to around to around 1,200 led to what was essentially a deadlock in the scheduling of new builds. New builds were stuck in a _New, waiting_ state indefinitely.

This [was fixed](https://github.com/openshift/origin/pull/12623) as of OCP 3.4.1, but it caused me to get more pro-active in the [pruning of artifacts](https://docs.openshift.org/latest/admin_guide/pruning_resources.html) within OpenShift.

I threw together a script and a playbook to deploy it. YMMV


Playbook to setup the first master to prune artifacts on a schedule. This will also create a `serviceaccount` in the `default` project with appropriate permissions to support image pruning.

```yaml
---
# file: prune-cron.yml
# run this on only the first master, as root

- hosts: masters[0]
  vars:
    default:
      cron_weekday: "1-5"
      cron_hour: 6
      cron_minute: "{{ 15 | random}}"
    prune_script: prune-artifacts
    prune_serviceaccount: prunebot
    prune_artifacts:
      builds:
        keep_complete: 5
        keep_failed: 5
        keep_younger: 60m
        cron_hour: 8
      deployments:
        keep_complete: 5
        keep_failed: 5
        keep_younger: 60m
        cron_hour: 7
      images:
        keep_younger: 60m
        keep_tag_revisions: 5
        cron_weekday: 1

  tasks:
    - name: check for service account
      command: "oc get sa {{ prune_serviceaccount }} -n default -o name"
      register: serviceaccount
      ignore_errors: true
      tags: sa

    - name: create service account
      command: "oc create sa {{ prune_serviceaccount }} -n default"
      when: "'{{ prune_serviceaccount }}' not in '{{ serviceaccount.stdout }}'"
      tags: sa

    - name: grant perms to service account
      command: "oc adm policy add-cluster-role-to-user system:image-pruner system:serviceaccount:default:{{ prune_serviceaccount }}"
      when: "'{{ prune_serviceaccount }}' not in '{{ serviceaccount.stdout }}'"
      tags: sa

    - name: install prune artifacts script
      copy:
        src: "bin/{{ prune_script }}"
        dest: "/usr/local/bin/{{ prune_script }}"
        owner: root
        group: root
        mode: 0755
      run_once: true
      tags: script

    - name: create prune crons per artifact type
      cron:
        name: "prune old {{ item.key }} artifacts"
        job: "/usr/local/bin/{{ prune_script }} --artifact '{{ item.key }}' --keep-complete '{{item.value.keep_complete | default(5)}}'  --keep-failed '{{item.value.keep_failed | default(5)}}'  --keep-younger '{{item.value.keep_younger | default('60m')}}' --keep-tag-revisions '{{item.value.keep_tag_revisions | default(5)}}'"
        user: root
        cron_file: "prune-{{ item.key }}"
        minute: "{{item.value.cron_minute | default(default.cron_minute)}}"
        hour: "{{item.value.cron_hour | default(default.cron_hour)}}"
        weekday: "{{item.value.cron_weekday | default(default.cron_weekday)}}"
        state: present
      with_dict: "{{ prune_artifacts }}"
      run_once: true
      tags: cron
```

Wrapper to call the `oc prune` command

```bash
#!/bin/bash
# prune-artifacts
# https://docs.openshift.org/latest/admin_guide/pruning_resources.html
# https://docs.openshift.com/container-platform/3.4/admin_guide/pruning_resources.html

KEEP_COMPLETE=5
KEEP_FAILED=5
KEEP_YOUNGER="60m"
KEEP_TAG_REVISIONS=3
PRUNE_SERVICEACCOUNT="prunebot"

USAGE="$0 --artifact <builds,deployments,images> --keep_complete <num> --keep_failed <num> --keep_younger <time> --keep-tag-revisions <num>"

while [[ $# -gt 1 ]]; do
  key="$1"

  case $key in
      --artifact)
        ARTIFACT="$2"
        shift
      ;;
      --keep-complete)
        KEEP_COMPLETE="$2"
        shift
      ;;
      --keep-failed)
        KEEP_FAILED="$2"
        shift
      ;;
      --keep-younger)
        KEEP_YOUNGER="$2"
        shift
      ;;
      --keep-tag-revisions)
        KEEP_TAG_REVISIONS="$2"
        shift
      ;;
  esac
  shift
done

LOGGER="logger -t prune-$ARTIFACT"

if [ -z "$ARTIFACT" ]; then
  echo "$USAGE"
  $LOGGER "$USAGE"
  exit 1
fi

if [ "$ARTIFACT" == "images" ]; then
  $LOGGER "pruning $ARTIFACT over $KEEP_YOUNGER, keep at least $KEEP_TAG_REVISIONS tag revisions as user $PRUNE_SERVICEACCOUNT"
  $LOGGER "oc --token=<token> adm prune $ARTIFACT --keep-tag-revisions=$KEEP_TAG_REVISIONS  --keep-younger-than=$KEEP_YOUNGER --confirm"
  oc --token=$(oc serviceaccounts get-token "$PRUNE_SERVICEACCOUNT") adm prune "$ARTIFACT" \
    --keep-tag-revisions="$KEEP_TAG_REVISIONS"  --keep-younger-than="$KEEP_YOUNGER" --confirm | $LOGGER

else
  $LOGGER "pruning $ARTIFACT over $KEEP_YOUNGER, keep at least $KEEP_COMPLETE and $KEEP_FAILED failed"

  artifact_count=$(oc adm prune $ARTIFACT \
    --orphans --keep-complete=$KEEP_COMPLETE --keep-failed=$KEEP_FAILED --keep-younger-than=$KEEP_YOUNGER 2>/dev/null | wc -l)
  if [ $? -eq 0 ]; then
    $LOGGER "count $artifact_count $ARTIFACT to delete"
    if [ "$artifact_count" -gt "0" ]; then
      $LOGGER "oc adm prune $ARTIFACT " \
        "--orphans --keep-complete=$KEEP_COMPLETE --keep-failed=$KEEP_FAILED --keep-younger-than=$KEEP_YOUNGER --confirm"
      oc adm prune "$ARTIFACT" \
        --orphans --keep-complete="$KEEP_COMPLETE" --keep-failed="$KEEP_FAILED" --keep-younger-than="$KEEP_YOUNGER" --confirm
    fi
  else
    $LOGGER "failed to count existing $ARTIFACT"
    exit 1
  fi
fi
```
