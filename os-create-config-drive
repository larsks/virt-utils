#!/bin/sh

# This will generate a openstack-style config drive image suitable for
# use with cloud-init.  You may optionally pass in an ssh public key
# (using the -k/--ssh-key option) and a user-data blog (using the
# -u/--user-data option).

md_api_versions='
2015-10-15
2016-06-30
2016-10-06
2017-02-22
'


usage () {
	echo "usage: ${0##*/}: [--ssh-key <pubkey>] [--vendor-data <file>] [--user-data <file>] [--hostname <hostname>] <imagename>"
}

ARGS=$(getopt \
	-o k:u:v:h:n: \
	--long help,hostname:,ssh-key:,user-data:,vendor-data:,network-data: -n ${0##*/} \
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
		-n|--network-data)
			network_data="$2"
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

if ! [ "$#" -eq 1 ]; then
	echo "ERROR: missing target filename" >&2
	exit 1
fi

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

mkdir -p "${config_dir}/openstack/latest"

if [ "$user_data" ] && [ -f "$user_data" ]; then
	echo "adding user data from $user_data"
	cp $user_data $config_dir/openstack/latest/user-data
fi

if [ "$vendor_data" ] && [ -f "$vendor_data" ]; then
	echo "adding vendor data from $vendor_data"
	cp $vendor_data "$config_dir/openstack/latest/vendor_data.json"
else
	echo "{}" > "$config_dir/openstack/latest/vendor_data.json"
fi

if [ "$network_data" ] && [ -f "$network_data" ]; then
	echo "adding network data from $network_data"
	cp $network_data "$config_dir/openstack/latest/network_data.json"
fi

cat > $config_dir/openstack/latest/meta_data.json <<-EOF
{
"uuid": "$uuid",
"hostname": "$hostname",
"name": "$hostname",
"launch_index": 0,
"availability_zone": "nova"
EOF

if [ "$ssh_key_data" ]; then
	cat >> $config_dir/meta_data.json <<-EOF
	,
	"keys": [
	{ "name": "default", "type": "ssh", "data": "$ssh_key_data" }
	],
	"public_keys": {
	  "default": "$ssh_key_data"
	}
	EOF
fi

echo "}" >> $config_dir/openstack/latest/meta_data.json

for v in $md_api_versions; do
	echo "setting up api version $v"
	cp -a "$config_dir/openstack/latest" "$config_dir/openstack/$v"
done

#PS1="debug> " bash --norc

echo "generating configuration image at $config_image"
if ! mkisofs -o $config_image -V cidata -r -J --quiet $config_dir; then
	echo "ERROR: failed to create $config_image" >&2
	exit 1
fi
chmod a+r $config_image

