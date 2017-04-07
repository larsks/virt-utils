#!/bin/sh

# This will generate a openstack-style config drive image suitable for
# use with cloud-init.  You may optionally pass in an ssh public key
# (using the -k/--ssh-key option) and a user-data blog (using the
# -u/--user-data option).

usage () {
	echo "usage: ${0##*/}: [--ssh-key <pubkey>] [--vendor-data <file>] [--user-data <file>] [--hostname <hostname>] <imagename>"
}

ARGS=$(getopt \
	-o k:u:v:h: \
	--long help,hostname:,ssh-key:,user-data:,vendor-data: -n ${0##*/} \
	-- "$@")

if [ $? -ne 0 ]; then
	usage >&2
	exit 2
fi

eval set -- "$ARGS"

while :; do
	case "$1" in
		--help)
			usage
			exit 0
			;;
		-k|--ssh-key)
			ssh_key="$2"
			shift 2
			;;
		-u|--user-data)
			user_data="$2"
			shift 2
			;;
		-v|--vendor-data)
			vendor_data="$2"
			shift 2
			;;
		-h|--hostname)
			hostname="$2"
			shift 2
			;;
		--)	shift
			break
			;;
	esac
done

config_image=$1
shift

if [ "$ssh_key" ] && [ -f "$ssh_key" ]; then
	echo "adding pubkey from $ssh_key"
	ssh_key_data=$(cat "$ssh_key")
fi

uuid=$(uuidgen)
if ! [ "$hostname" ]; then
	hostname="$uuid"
fi

trap 'rm -rf $config_dir' EXIT
config_dir=$(mktemp -t -d configXXXXXX)

if [ "$user_data" ] && [ -f "$user_data" ]; then
	echo "adding user data from $user_data"
	cp $user_data $config_dir/user-data
else
	touch $config_dir/user-data
fi

if [ "$vendor_data" ] && [ -f "$vendor_data" ]; then
	echo "adding vendor data from $vendor_data"
	cp $vendor_data $config_dir/vendor-data
fi

cat > $config_dir/meta-data <<-EOF
instance-id: $uuid
hostname: $hostname
local-hostname: $hostname
EOF

if [ "$ssh_key_data" ]; then
	cat >> $config_dir/meta-data <<-EOF
	public-keys:
	  - |
	    $ssh_key_data
	EOF
fi

#PS1="debug> " bash --norc

echo "generating configuration image at $config_image"
if ! mkisofs -o $config_image -V cidata -r -J --quiet $config_dir; then
	echo "ERROR: failed to create $config_image" >&2
	exit 1
fi
chmod a+r $config_image

