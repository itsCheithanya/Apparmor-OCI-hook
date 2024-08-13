# Apparmor-OCI-hook-LFX prerequisite task
The first step required is actually configuring the hooks definition location by editing the ```/etc/containers/containers.conf``` if its exists in you system or this ```/usr/share/containers/containers.conf``` file:
```
[engine]
hooks_dir = ["/etc/containers/oci/hooks.d"]
```
Now create the OCI hook definition like this apparmor-hook.json in the ```/etc/containers/oci/hooks.d``` directory
```
{
    "version": "1.0.0",
    "hook": {
    "path": "/usr/local/bin/oci-apparmor-hook"
    },
    "when": {
    "always": true
    },
    "stages": ["precreate"]
    }
```
Create an apparmor profile like this "test-profile-etc",
The key line here is deny /etc/** wl, which blocks write access to /etc and any subdirectories. We'll write this profile to ```/etc/apparmor.d/test-profile-etc``` and then use the command ```$ sudo apparmor_parser -r etc/apparmor.d/test-profile-etc``` to load it into the kernel 

```
#include <tunables/global>
profile test-profile-etc flags=(attach_disconnected,mediate_deleted) {
#include <abstractions/base>
file,
deny /etc/** wl,
}
```
Now create the hook binary oci-apparmor-hook in the path location mentioned in the hook definition JSON and update your apparmor profile name in the hook binary code in the variable ``` APPARMOR_PROFILE```
```
#!/bin/bash
echo "Running oci-apparmor-hook" >> /var/log/oci-apparmor-hook.log

CONTAINERS_CONF_PATH="/usr/share/containers/containers.conf"

if [ ! -f "$CONTAINERS_CONF_PATH" ]; then
    echo "Error: $CONTAINERS_CONF_PATH does not exist." >&2
    exit 1
fi

if grep -q "^apparmor_profile" "$CONTAINERS_CONF_PATH"; then
    sed -i "s|^apparmor_profile.*|apparmor_profile = \"$APPARMOR_PROFILE\"|g" "$CONTAINERS_CONF_PATH"
else
    sed -i "/^\[containers\]/a apparmor_profile = \"$APPARMOR_PROFILE\"" "$CONTAINERS_CONF_PATH"
fi

CONTAINER_CONFIG=$(cat)

# Set the AppArmor profile to be used
APPARMOR_PROFILE="test-profile-etc"

MODIFIED_CONFIG=$(echo "$CONTAINER_CONFIG" | jq --arg profile "$APPARMOR_PROFILE" '
    .process.apparmorProfile = $profile |
    .annotations."io.podman.annotations.apparmor" = $profile |
    .hostConfig.SecurityOpt += ["apparmor=" + $profile]
')

if [ $? -ne 0 ]; then
    echo "Error: Failed to modify configuration JSON" >> /var/log/oci-apparmor-hook.log
    exit 1
fi

echo "$MODIFIED_CONFIG"

echo "AppArmor profile $APPARMOR_PROFILE applied successfully and annotations updated." >> /var/log/oci-apparmor-hook.log

exit 0


```
Now run a podman contiainer like this
``` $ sudo podman run --name foobar --detach --rm -ti fedora ``` and you should see that according to your apparmor policy you should have it being enforced in the container,you can verify the apparmor profile by getting inside the container using
``` $ sudo podman exec  -it foobar bash ```  and running the command ``` $ cat /proc/self/attr/current ``` which shows the running apparmor profile

![Screenshot from 2024-08-12 23-07-09](https://github.com/user-attachments/assets/58b95e74-5dbc-47d6-bedb-963589cf2c15)
