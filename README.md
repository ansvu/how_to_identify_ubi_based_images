# How to Identify UBI Base Images
There was a scenario where we needed to determine if over 100 container images were Red Hat UBI-based or third-party images. To check if there were any non-UBI images, such as third-party images used in Telco CNFs stored on a Red Hat private Quay registry server, we had to manually inspect each image or use tools to do so. This was a time-consuming process, taking several minutes for each image. 

To simplify and streamline the process, I created a knowledge base using shell script and REST API to identify the images and output the results in a CSV file or on the screen, reducing the time to just 2-3 minutes for 100 images.

**Note:** If you have more 10-200 images are on Quay Private Registry Server, then it worths to use KB as your reference to start.
Also, this KB only aim to identify if container image is UBI or Non-UBI bases but same approach can be used for other purpose, where you can update the script to meet your needs. 

# Pre-requisite
- Based on Quay 3.8 API version 
- Create OAuth Applications Token [How to create Oauth App TOken](https://access.redhat.com/documentation/en-us/red_hat_quay/3/html/red_hat_quay_api_guide/using_the_red_hat_quay_api#create_oauth_access_token)
- Skopeo tool [How to install skopeo](https://github.com/containers/skopeo/blob/main/install.md)  
  `sudo dnf -y install skopeo on RHEL8`
- Make sure connectivity to Quay register server reachable
- Expect to use Curl to test out REST API first

# Instructions and Action
## ShellScript Usage
```shellSession
./check_ubi_base_images_v1-KB.sh
------------------------------------------------------------------------------------------------------------------------
Usage: ./check_ubi_base_images_v1-KB.sh -rn|--repo-ns <org_name|user_name> -cp|--cnf-prefix <common_image_name> -t|--tag-type <name|digest> -lt|--log-type <raw|csv> -ft|--filter <filter_me>
Usage: ./check_ubi_base_images_v1-KB.sh [-h | --help]
Usage ex1: ./check_ubi_base_images_v1-KB.sh -rn ava -cp "global-|specific" -t name -lt csv -ft "existed_image|tested_image"
Usage ex2: ./check_ubi_base_images_v1-KB.sh --repo-ns avareg_5gc --cnf-prefix global- --tag-type name --log-type csv
Usage ex3: ./check_ubi_base_images_v1-KB.sh --repo-ns avareg_5gc --cnf-prefix global-

Note: tag-type and log-type can be excluded from argument


    -rn|--repo-ns        :  An organization or user name e.g avareg_5gc or avu
    -cp|--cnf-prefix     :  Is CNF image prefix e.g. global-amf-rnic or using wildcard
                            It also uses more one prefix e.g. "global|non-global"

    -t|--tag-type        :  Image Tag Type whether it requires to use tag or digest name, preferred tag name
                            If name or digest argument is omitted it uses default tag name

    -lt|--log-type       :  raw type will print result output as json to console
                            csv type will dump to two csv files 1. non-ubi 2. ubi

    -ft|--filter         :  If you want to exclude images or unwanted e.g. chartrepo or tested-images, then
                            pass to script argument like this:
                            ./check_ubi_base_images_v1-KB.sh -rn ava -cp global- -t name -lt csv -ft "existed_image|tested_image"
    
------------------------------------------------------------------------------------------------------------------------
```
### Script Contents
```bash
#!/bin/bash

quay_oauth_api_key="xxxxxxxxxxxxxxxxxxxxxx"
quay_registry_domain="quay.avareg.bos2.lab"
non_ubi_image_list_file="3rdparty_image_list.csv"
ubi_image_list_file="ubi_image_list.csv"

print_help() {
    echo "------------------------------------------------------------------------------------------------------------------------"
    echo "Usage: $0 -rn|--repo-ns <org_name|user_name> -cp|--cnf-prefix <common_image_name> -t|--tag-type <name|digest> -lt|--log-type <raw|csv> -ft|--filter <filter_me>"
    echo "Usage: $0 [-h | --help]"
    echo "Usage ex1: $0 -rn ava -cp \"global-|specific\" -t name -lt csv -ft \"existed_image|tested_image\""
    echo "Usage ex2: $0 --repo-ns avareg_5gc --cnf-prefix global- --tag-type name --log-type csv"
    echo "Usage ex3: $0 --repo-ns avareg_5gc --cnf-prefix global-"
    echo ""
    echo "Note: tag-type and log-type can be excluded from argument"
    echo ""
    echo "
    -rn|--repo-ns        :  An organization or user name e.g avareg_5gc or avu
    -cp|--cnf-prefix     :  Is CNF image prefix e.g. global-amf-rnic or using wildcard
                            It also uses more one prefix e.g. \"global|non-global\"

    -t|--tag-type        :  Image Tag Type whether it requires to use tag or digest name, preferred tag name
                            If name or digest argument is omitted it uses default tag name

    -lt|--log-type       :  raw type will print result output as json to console
                            csv type will dump to two csv files 1. non-ubi 2. ubi

    -ft|--filter         :  If you want to exclude images or unwanted e.g. chartrepo or tested-images, then
                            pass to script argument like this:
                            $0 -rn ava -cp global- -t name -lt csv -ft \"existed_image|tested_image\"
    "
    echo "------------------------------------------------------------------------------------------------------------------------"
    exit 0
}
for i in "$@"; do
    case $i in
    -rn | --repo-ns)
        if [ -n "$2" ]; then
            REPO_NS="$2"
            shift 2
            continue
        fi
        ;;
    -cp | --cnf-prefix)
        if [ -n "$2" ]; then
            CNF_PREFIX="$2"
            shift 2
            continue
        fi
        ;;
    -t | --tag-type)
        if [ -n "$2" ]; then
            TAG_TYPE="$2"
            shift 2
            continue
        fi
        ;;
    -lt | --log-type)
        if [ -n "$2" ]; then
            LOG_TYPE="$2"
            shift 2
            continue
        fi
        ;;
    -ft | --filter)
        if [ -n "$2" ]; then
            FILTER="$2"
            shift 2
            continue
        fi
        ;;
    -h | -\? | --help)
        print_help
        shift #
        ;;
    *)
        # unknown option
        ;;
    esac
done

#Note: tag-type and log-type can be excluded from argument#
if [[ "$REPO_NS" == "" || "$CNF_PREFIX" == "" ]]; then
    print_help
fi

if [[ "$TAG_TYPE" == "" ]]; then
    TAG_TYPE="name"
fi

#if filter arg is empty, then we will filter chartrepo
if [[ "$FILTER" == "" ]]; then
    FILTER="chartrepo"
fi

if [[ "$LOG_TYPE" == "" ]]; then
    LOG_TYPE="csv"
fi
echo "Please be patient while checking images..."

#Get all images based user's criteria and filters from QAUY via REST API#
readarray -t ImageLists <<<$(curl --silent -X GET -H "Authorization: Bearer ${quay_oauth_api_key}" "https://${quay_registry_domain}/api/v1/repository?namespace=${REPO_NS}" | jq -r '.repositories[].name' | egrep ${CNF_PREFIX} | egrep -v ${FILTER})

#Remove old csv flies#
rm -f ${ubi_image_list_file} ${non_ubi_image_list_file}

#Add header to csv files#
printf "%s\n" "Image Name,OS Type,RHEL Version" | tee ${ubi_image_list_file} >/dev/null
printf "%s\n" "Image Name,OS Type" | tee ${non_ubi_image_list_file} >/dev/null

for ((j = 0; j < ${#ImageLists[*]}; j++)); do
    echo "Checking follow image_name for Non-UBI Based: ${ImageLists[$j]}"

    image_url="https://${quay_registry_domain}/api/v1/repository/${REPO_NS}/${ImageLists[$j]}"
    if [[ "${TAG_TYPE}" == "name" ]]; then
        tag_type_flag=".name + \":\" + .tags[].name"
    else # digest
        tag_type_flag=".name + \"@\" + .tags[].manifest_digest"
    fi

    image_details=$(curl --silent -X GET -H "Authorization: Bearer ${quay_oauth_api_key}" "${image_url}" | jq -r "$tag_type_flag")
    tag=$(echo $image_details | cut -d ':' -f2)
    inspect_url="docker://${quay_registry_domain}/${REPO_NS}/${ImageLists[$j]}:$tag"
    if [[ "${LOG_TYPE}" == "raw" ]]; then
        skopeo inspect --config $inspect_url | jq '.config.Labels | { name: .name, vendor: .vendor, architecture: .architecture, version: .version }'
    else #csv
        skopeo_json_output=$(skopeo inspect --config $inspect_url | jq '.config.Labels | { name: .name, vendor: .vendor, architecture: .architecture, version: .version }')
        if [[ "$skopeo_json_output" == *"null"* ]]; then
            printf "%s\n" "${ImageLists[$j]},non-ubi" >>$non_ubi_image_list_file
        elif [[ "$skopeo_json_output" == *"ubi"* ]]; then
            rhel_version=$(echo $skopeo_json_output | jq '.version' | sed 's/\"//g')
            ubi_version=$(echo $skopeo_json_output | jq '.name' | sed 's/\"//g')
            printf "%s\n" "${ImageLists[$j]},${ubi_version},${rhel_version}" >>$ubi_image_list_file
        else #Other OS Type
            printf "%s\n" "$skopeo_json_output"
        fi
    fi
done
```
---
**Note**: On top of the script following two lines must update manually beside those argument the script itself when using it  
- Quay Oauth App Token replace with xxxxxxxxxxxxxxxxxxx
- Quay Registry Doamin or IP Address
```shellSession
quay_oauth_api_key="xxxxxxxxxxxxxxxxxxxxxx"
quay_registry_domain="quay.avareg.bos2.lab"
```
### Start The Test Run to Dump To CSV  Files
```diff
+ ./check_ubi_base_images_v1-KB.sh --repo-ns ava_5gc --cnf-prefix "global-|rel-5gcore" --tag-type name --log-type csv
Please be patient while checking images...
Checking follow image_name for Non-UBI Based: rel-ava/global-av-busybox
Checking follow image_name for Non-UBI Based: rel-5gcore/av-necc
Checking follow image_name for Non-UBI Based: rel-5gcore/av-uds
Checking follow image_name for Non-UBI Based: rel-test/global-av-fluentd
Checking follow image_name for Non-UBI Based: rel-5gcore/av-ipds
Checking follow image_name for Non-UBI Based: rel-test/global-av-alpine
...
```
- **UBI Image List**  
ubi_image_list.csv:
```shellSession
Image Name,OS Type,RHEL Version
rel-5gcore/av-uds,ubi8,8.5
rel-5gcore/av-necc,ubi8,8.6
rel-5gcore/av-ipds,ubi8,8.6
...
```
- **Non UBI Base Image List**  
3rdparty_image_list.csv:
```shellSession
Image Name,OS Type
rel-ava/global-av-busybox,non-ubi
rel-test/global-av-alpine,non-ubi
rel-test/global-av-fluentd,non-ubi
```
### Start The Test Run with RAW On Screen
```diff
+./check_ubi_base_images_v1-KB.sh --repo-ns ava_5gc --cnf-prefix "global-|rel-5gcore" --tag-type name --log-type raw
Please be patient while checking images...  
Checking follow image_name for Non-UBI Based: rel-ava/global-av-busybox
{
  "name": null,
  "vendor": null,
  "architecture": null,
  "version": null
}
Checking follow image_name for Non-UBI Based: rel-5gcore/av-ipds
{
  "name": "ubi8",
  "vendor": "Red Hat, Inc.",
  "architecture": "x86_64",
  "version": "8.6"
}
Checking follow image_name for Non-UBI Based: rel-5gcore/av-necc
{
  "name": "ubi8",
  "vendor": "Red Hat, Inc.",
  "architecture": "x86_64",
  "version": "8.6"
}
Checking follow image_name for Non-UBI Based: rel-test/global-av-fluentd
{
  "name": null,
  "vendor": null,
  "architecture": null,
  "version": null
}
```
**Note:** If Vendors/Partners are using UBI Dockerfile from RedHat, then mostly like the Labels must be present and defined other data accordingly.

# Conclusion
To summarize the purpose of this KB, the manual inspection of over 100 container images to determine their Red Hat UBI-based or third-party status was a time-consuming process. To streamline this process, that is why this knowledge base was created using a shell script, skopeo and REST API, reducing the time to just 2-3 minutes for 100 images. This knowledge base serves as a reference point if you plan to check 100's of images on a Quay private registry server and can be updated to meet different needs.
