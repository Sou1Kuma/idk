import json
import os
import requests
import zipfile
import subprocess
import time
from datetime import datetime
from requests.exceptions import ConnectionError, Timeout
from http.client import IncompleteRead
import uuid
import platform
import colorama
from colorama import Fore, Style
import PlayFab
from PlayFab import LoginWithCustomId, Search, GetEntityToken, Search_name
import re
import shutil
import skin
import tsv
import dlc

def clear_console():
    os_name = platform.system()
    if os_name == 'Windows':
        os.system('cls')
    else:
        os.system('clear')

clear_console()

# Initialize colorama
colorama.init()

title = colorama.Fore.YELLOW + """
███████╗██████╗ ███████╗███████╗
██╔════╝██╔══██╗██╔════╝██╔════╝
█████╗  ██████╔╝█████╗  █████╗  
██╔══╝  ██╔══██╗██╔══╝  ██╔══╝  
██║     ██║  ██║███████╗███████╗
╚═╝     ╚═╝  ╚═╝╚══════╝╚══════╝
""" + colorama.Style.RESET_ALL

print(title)
print()

auth_token = None

def download_progress(downloaded, total_size):
    percent = int((downloaded / total_size) * 100)
    print(f"\rDownloading: {percent}%", end="", flush=True)


def search_skins(pack_type):  

    if pack_type == "--skin":
        url = "https://www.minecraft.net/bin/minecraft/productmanagement.productsinfobytype.json?type=skinpack&locale=en-us&limit=300&skip=0"
    elif pack_type == "--allskin":
        print("It could take a few minutes to fetch the whole list, be patient.")
        url = "https://www.minecraft.net/bin/minecraft/productmanagement.productsinfobytype.json?type=skinpack&locale=en-us&limit=98000&skip=0"

    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/123.0.0.0 Safari/537.36",
    }
    
    try:
        response = requests.get(url, headers=headers)
        response.raise_for_status()
        
        data = response.json()
        if data:
            return data
        else:
            print("No result found.")
    except requests.RequestException as e:
        print(f"Failed to fetch data. (Try again) Error: {e}")
    
    return None

# Extract ID from URL
def extract_id_from_url(url):
    pattern = r'id=([0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12})'
    match = re.search(pattern, url)
    if match:
        return match.group(1)
    else:
        return None

def check_custom_id(custom_ids):
    # Read all lines from the file once
    with open("keys.tsv", "r") as keys_file:
        lines = keys_file.readlines()
    if isinstance(custom_ids, str):  # Handle single ID
        custom_ids = {custom_ids}
    elif isinstance(custom_ids, list):  # Handle list of IDs
        custom_ids = set(custom_ids)

    # Iterate through each line and check if any ID matches
    for line in lines:
        for id in custom_ids:
            if id in line:
                return True

    return False

def display_help():
    help_message = f"""
    Instructions usage commands:

    {Fore.YELLOW}--dlc{Style.RESET_ALL} / get the list of worldtemplate available in keys

    {Fore.YELLOW}--addon{Style.RESET_ALL} / get the list of all addons

    {Fore.YELLOW}--skin{Style.RESET_ALL} / get the list of the first 300 skins

    {Fore.YELLOW}--mashup{Style.RESET_ALL} / get the list of all mashup

    {Fore.YELLOW}--texture{Style.RESET_ALL} / get the list of all texture packs

    {Fore.YELLOW}--persona (name){Style.RESET_ALL} / search for persona items,
        example: --persona mcc crown
    
    {Fore.YELLOW}--capes{Style.RESET_ALL} / get the list of all capes

    {Fore.YELLOW}--allskin{Style.RESET_ALL} / get the list of all skinpacks,
        note: there are 22k skins, it can take a while 
        depending on your connection

    {Fore.YELLOW}--newest{Style.RESET_ALL} / display first 300 latest items 

    {Fore.YELLOW}--hidden{Style.RESET_ALL} / display first 300 unlisted items

    {Fore.YELLOW}--allhidden{Style.RESET_ALL} / display all the unlisted items

    {Fore.YELLOW}--list{Style.RESET_ALL} / display the list of the items present on the keys.tsv

    {Fore.YELLOW}--of{Style.RESET_ALL} / activate offline search,
        its gonna display only the items in keys

    {Fore.YELLOW}--all{Style.RESET_ALL} / select every numbers on the list 
        note: this is gonna download everything
        present on the list

    {Fore.YELLOW}--reload{Style.RESET_ALL} / force reload keys.tsv-list.txt update

    {Fore.YELLOW}--new{Style.RESET_ALL} / display new added items

    {Fore.YELLOW}--extra{Style.RESET_ALL} / display extra features

    {Fore.YELLOW}--exit{Style.RESET_ALL} / exit the program

"""
    print(help_message)

def display_extra():
    # Check the current setting for DisplayLastModifiedDate
    with open("settings.json", "r") as file:
        settings = json.load(file)
    display_last_modified_date = settings.get("DisplayLastModifiedDate", "False")

    status = "ON" if display_last_modified_date == "True" else "OFF"
    extra_message = f"""
    Extra features:

    Display Last Modified Date: {Fore.GREEN if status == "ON" else Fore.RED}{status}{Style.RESET_ALL}
    {Fore.YELLOW}--date (ON/OFF){Style.RESET_ALL} / Show the last modified date on DLC
        example: --date ON
    """
    print(extra_message)


def update_keys():
    global global_new_lines
    try:
        tsv.force_update_keys()
        global_new_lines, _ = tsv.check_dlc_list()
    except Exception as e:
        log_error(None, e)

def login():
    global auth_token
    print("API Login...", end="", flush=True)
    response = LoginWithCustomId()
    if 'PlayFabId' in response:
        auth_token = GetEntityToken(response['PlayFabId'], 'master_player_account')
        print("\r" + " " * len("API Login...") + "\r", end="", flush=True)
    else:
        print("\rLogin failed.", end="\n", flush=True)
        return False
    return True

def perform_search(query, orderBy, select, top, skip, search_type, search_term):
    global auth_token
    if not auth_token:
        if not login():
            return None
    try:
        return Search_name(query=query, orderBy=orderBy, select=select, top=top, skip=skip, search_type=search_type, search_term=search_term)
    except Exception as e:
        if 'Unauthorized' in str(e):
            print("Session expired. Re-authenticating...")
            if login():
                return Search_name(query=query, orderBy=orderBy, select=select, top=top, skip=skip, search_type=search_type, search_term=search_term)
        print(f"An error occurred: {e}")
        return None

def load_settings(file_path="settings.json"):
    try:
        with open(file_path, "r") as file:
            return json.load(file)
    except (FileNotFoundError, json.JSONDecodeError):
        return {}

def update_settings(file_path="settings.json", key="DisplayLastModifiedDate", value="False"):
    settings = load_settings(file_path)
    settings[key] = value
    with open(file_path, "w") as file:
        file.write(json.dumps(settings, separators=(",", ":")))

def display_items(filtered_items):
    settings = load_settings()
    display_last_modified = settings.get("DisplayLastModifiedDate", "False") == "True"
    
    print()
    for idx, item in enumerate(filtered_items, start=1):
        title_en_us = item.get("Title", {}).get("en-US", item.get("Title", {}).get("en-us", "Title not available"))
        creator_name = item.get("DisplayProperties", {}).get("creatorName", "Creator name not available")
        
        last_updated = ""
        if display_last_modified:
            last_updated_iso = item.get("LastModifiedDate")
            if last_updated_iso:
                try:
                    last_updated_dt = datetime.fromisoformat(last_updated_iso.replace("Z", "+00:00"))
                    last_updated = last_updated_dt.strftime("%B %d, %Y, %H:%M")
                except ValueError:
                    last_updated = "Invalid date format"

        pack_type = "SkinPack" if "skinpack" in item.get("Tags", []) else "DLC"
        pack_type = "Addon" if "addon" in item.get("Tags", []) else pack_type
        pack_type = "TexturePack" if "resourcepack" in item.get("Tags", []) else pack_type
        pack_type = "Mashup" if "mashup" in item.get("Tags", []) else pack_type
        pack_type = "Persona" if "PersonaDurable" in item.get("ContentType", []) else pack_type
        checkbox = (Fore.GREEN + "[✓]" + Fore.RESET if pack_type == "SkinPack" or pack_type == "Persona" 
                    else (Fore.GREEN + "[✓]" + Fore.RESET if check_custom_id(item["Id"]) 
                          else Fore.RED + "[X]" + Fore.RESET))
        
        # Print the output based on whether last_updated is enabled
        if display_last_modified:
            print(f"{idx}) {title_en_us} ( {creator_name} ) {last_updated} - {pack_type} {checkbox}")
        else:
            print(f"{idx}) {title_en_us} ( {creator_name} ) - {pack_type} {checkbox}")
    print()

def read_list():
    try:
        with open("list.txt", "r") as file:
            lines = file.readlines()
        return lines
    except FileNotFoundError:
        print("Error: 'list.txt' not found.")
        return []

def display_keys_items(filter_keyword=None):
    lines = read_list()
    if filter_keyword:
        # Convert the keyword and each line to lowercase for case-insensitive search
        filter_keyword_lower = filter_keyword.lower()
        filtered_lines = [line for line in lines if filter_keyword_lower in line.lower()]
    else:
        filtered_lines = lines

    print()    
    for idx, line in enumerate(filtered_lines, start=1):
        display_line = re.sub(r'\s*[\da-fA-F-]{36}$', '', line).strip()
        print(f"{idx}) {display_line} {Fore.GREEN}[✓]{Fore.RESET}")
    print()
    return filtered_lines
EXIT_COMMAND = "__EXIT__"
def get_custom_id():
    global global_new_lines
    while True:
        try:
            user_input = input("(type --help for the command instructions)\nEnter the NAME, UUID or URL: ").strip()
        except EOFError: # This issue happens only on Termux on the first start
            print("\nEOFError encountered. Please restart the app")
            return EXIT_COMMAND
        

        if user_input.lower() == "--exit":
            print("Exiting the program.")
            return EXIT_COMMAND
        
        if user_input.lower() in ["--help", "--reload", "--list", "--dlc", "--skin", "--allskin", "--of", "--exit", "--new", "--extra", "--date"]:
            if user_input.lower() == "--reload":
                update_keys()
                continue
            elif user_input.lower() == "--help":
                display_help()
                continue
            elif user_input.lower() == "--extra":
                display_extra()
                continue
            elif user_input.lower() == "--new":
                if global_new_lines:
                    print("Added items in keys:")
                    print()
                    for line in global_new_lines:
                        stripped_line = re.sub(r'\s*[\da-fA-F-]{36}$', '', line).strip()
                        print(stripped_line)
                    print()
                else:
                    print("No new items added.")
                continue
            elif user_input.lower().startswith("--date"):
                command_parts = user_input.split()
                if len(command_parts) > 1 and command_parts[1].upper() in ["ON", "OFF"]:
                    update_settings(key="DisplayLastModifiedDate", value="True" if command_parts[1].upper() == "ON" else "False")
                    print(f"Display Last Modified Date is now {'ON' if command_parts[1].upper() == 'ON' else 'OFF'}.")
                else:
                    print("Invalid usage of --date. Use --date ON or --date OFF.")
                continue


            if user_input.lower() in ["--list", "--dlc"]:
                items = display_keys_items(filter_keyword="DLC" if user_input.lower() == "--dlc" else None)
                if not items:
                    continue
            elif user_input.lower() == "--of":
                search_term = input("Enter search term for offline search: ").strip()
                items = display_keys_items(filter_keyword=search_term)
                if not items:
                    continue
            elif user_input.lower() in ["--skin", "--allskin"]:
                items = search_skins(user_input.lower())
                if not items:
                    continue
                display_items(items)
            
            
            while True:
                selected_numbers = input("(Type 'R' to retry the search)\nSelect the number(s) separated by commas: ")
                if selected_numbers.lower() == 'r':
                    break
                elif selected_numbers.lower() == '--all':
                    if user_input.lower() in ["--list", "--dlc", "--of"]:
                        return {"ids": [re.search(r'[\da-fA-F-]{36}', item).group() for item in items], "use_playfab": True}
                    else:
                        return {"ids": [item.get("id") or item.get("Id") for item in items], "use_playfab": True}
                else:
                    selected_ids = process_selected_numbers(selected_numbers, items)
                    if selected_ids:
                        if user_input.lower() in ["--list", "--dlc", "--of"]:
                            return {"ids": [re.search(r'[\da-fA-F-]{36}', items[int(num)-1]).group() for num in selected_numbers.split(',')], "use_playfab": True}
                        else:
                            return {"ids": selected_ids, "use_playfab": True}
                    print("Invalid selection. Please enter valid number(s).")
            continue

        elif user_input.lower().startswith("--date"):
            command_parts = user_input.split()
            if len(command_parts) > 1 and command_parts[1].upper() in ["ON", "OFF"]:
                status = command_parts[1].upper()
                update_settings(key="DisplayLastModifiedDate", value="True" if status == "ON" else "False")
                
                color = Fore.GREEN if status == "ON" else Fore.RED
                print(f"Display Last Modified Date is now {color}{status}{Style.RESET_ALL}.")
            else:
                print("Invalid usage of --date. Use --date ON or --date OFF.")
            continue

        elif "id=" in user_input:
            return {"ids": extract_id_from_url(user_input), "use_playfab": True}
        
        elif all(re.match(r'[\da-fA-F-]{36}', uuid.strip(), re.I) for uuid in user_input.split(',')):
            return {"ids": [uuid.strip() for uuid in user_input.split(',')], "use_playfab": True}
        
        else:
            search_command = "name"
            search_term = user_input
            if user_input.startswith('--'):
                command_parts = user_input.split()
                search_command = command_parts[0][2:]
                search_term = ' '.join(command_parts[1:]) if len(command_parts) > 1 else None

            if search_command not in ["name", "texture", "mashup", "addon", "persona", "capes", "hidden", "allhidden", "newest"]:
                print(f"Invalid command")
                continue
            
            data = perform_search(query="", orderBy="creationDate DESC", select="contents", top=300, skip=0, search_term=search_term, search_type=search_command)
            items = data.get("Items", []) if isinstance(data, dict) else data

            if items is None:
                continue

            items = [item for item in items if all(term in item.get("Title", {}).get("en-US", "").lower() for term in (search_term.lower().split() if search_term else []))]
            
            if not items:
                print("No result.")
                continue

            while True:
                display_items(items)
                selected_numbers = input("(Type 'R' to retry the search)\nSelect the number(s) separated by commas: ")
                if selected_numbers.lower() == 'r':
                    break
                elif selected_numbers.lower() == '--all':
                    return {"items": items, "use_playfab": False}
                else:
                    selected_ids = process_selected_numbers(selected_numbers, items)
                    if selected_ids:
                        return {"items": [item for item in items if item.get("id") in selected_ids or item.get("Id") in selected_ids], "use_playfab": False}
                    print("Invalid selection. Please enter valid number(s).")

def process_selected_numbers(selected_numbers, items):
    selected_numbers = selected_numbers.replace(" ", "").split(',')
    selected_ids = []
    for number in selected_numbers:
        try:
            selected_idx = int(number) - 1
            if 0 <= selected_idx < len(items):
                if isinstance(items[selected_idx], str):
                    selected_id = re.search(r'[\da-fA-F-]{36}', items[selected_idx]).group()
                else:
                    selected_id = items[selected_idx].get("id") or items[selected_idx].get("Id")
                if selected_id:
                    selected_ids.append(selected_id)
                else:
                    return None
            else:
                return None
        except (ValueError, AttributeError):
            return None
    return selected_ids

def download_and_process_zip(zip_url, output_folder, retries=3, timeout=160):
    retry_count = 0
    
    while retry_count < retries:
        try:
            response = requests.get(zip_url, timeout=timeout, headers={"User-Agent": "libhttpclient/1.0.0.0"}, stream=True)
            response.raise_for_status()

            total_size = int(response.headers.get('content-length', 0))
            downloaded_size = 0

            url_parts = zip_url.split("/")
            zip_filename = url_parts[-1]

            random_folder_name = uuid.uuid4().hex
            pack_folder = os.path.join(output_folder, random_folder_name)

            os.makedirs(pack_folder, exist_ok=True)

            # Download the ZIP file
            zip_file_path = os.path.join(pack_folder, zip_filename)
            with open(zip_file_path, "wb") as f:
                for chunk in response.iter_content(chunk_size=8192):
                    f.write(chunk)
                    downloaded_size += len(chunk)
                    download_progress(downloaded_size, total_size)

            print("\rDownload completed")

            print("\rExtracting zip file...")
            extracted_pack_folders = []  # List to store the folder paths of the extracted packages
            with zipfile.ZipFile(zip_file_path, 'r') as zip_ref:
                for name in zip_ref.namelist():
                    if name.endswith('.zip'):
                        nested_zip_file_path = os.path.join(pack_folder, name)
                        nested_pack_folder = os.path.join(pack_folder, os.path.splitext(name)[0])
                        if not os.path.exists(nested_pack_folder):
                            os.makedirs(nested_pack_folder)
                        zip_ref.extract(name, pack_folder)
                        with zipfile.ZipFile(nested_zip_file_path, 'r') as nested_zip_ref:
                            nested_zip_ref.extractall(nested_pack_folder)
                        os.remove(nested_zip_file_path)  # Remove file zip after extraction

                        # Add the extracted package name and folder path to the list
                        extracted_pack_folders.append((os.path.splitext(name)[0], nested_pack_folder))

            # Remove downloaded zip file
            os.remove(zip_file_path)
            print("\rZip file extracted")

            return extracted_pack_folders
        
        except (ConnectionError, Timeout, IncompleteRead) as e:
            print(f"\nFailed to download ZIP file: {e}")
            retry_count += 1
            print(f"Retrying... ({retry_count}/{retries})")
            time.sleep(5)  # Wait before retrying
        
        except zipfile.BadZipFile:
            print("The downloaded file is not a valid ZIP archive.")
            return None
        
        except Exception as e:
            print(f"\nAn error occurred: {e}")
            return None

    print("Exceeded maximum retries. Download failed.")
    return None

def check_for_addon(folder_path):
    manifest_path = os.path.join(folder_path, "manifest.json")
    if os.path.exists(manifest_path):
        with open(manifest_path, 'r') as manifest_file:
            manifest_data = json.load(manifest_file)

        is_manifest_addon = manifest_data.get("metadata", {}).get("product_type") == "addon"
        return is_manifest_addon
    else:
        return False
    
def data_uuid(folder_path):
    manifest_path = os.path.join(folder_path, "manifest.json")
    if os.path.exists(manifest_path):
        with open(manifest_path, 'r') as manifest_file:
            manifest_data = json.load(manifest_file)
            first_uuid = manifest_data.get("header", {}).get("uuid")
        return first_uuid
    return None

def log_error(first_uuid, e):
    error_log_file = 'error_log.txt'
    if first_uuid:
        error_message = f"Error processing pack: {first_uuid} - Error: {str(e)} (please report it)"
    else:
        error_message = f"Error: {str(e)} (please report it)"
    
    print(error_message)
    with open(error_log_file, 'a') as error_file:
        error_file.write(error_message + '\n')

def remove_extracted_folder(extracted_folder):
    print("removing extracted folder...")
    parent_folder = os.path.dirname(extracted_folder)
    shutil.rmtree(parent_folder, ignore_errors=True)
    print("extracted folder removed")

def main():
    output_folder = 'packs'
    global auth_token
    global global_new_lines
    try:
        tsv.update_keys()
        global_new_lines, _ = tsv.check_dlc_list()
    except Exception as e:
        log_error(None, e)
        pass

    if not login():
        print("Unable to authenticate.")
        return
    
    
    while True:
        result = get_custom_id()
        if result == EXIT_COMMAND:
            #username = subprocess.check_output(['whoami']).decode().strip()
            #subprocess.run(['killall', '-9', '-v', '-g', '-u', username]) - kill session for Termux
            break

        if result:
            if result.get("use_playfab"):
                resultsDict = PlayFab.main(result["ids"])
                skin_urls = []
                other_urls = []
                for entry in resultsDict.values():
                    title = entry.get("Title", {}).get("en-US", "")
                    creator_name = entry.get("DisplayProperties", {}).get("creatorName", "")
                    for content in entry["Contents"]:
                        if content.get("Type") == "skinbinary" or content.get("Type") == "personabinary":
                            skin_urls.append(content["Url"])
                        elif check_custom_id(result["ids"]):
                            other_urls.append(content["Url"])
                        else:
                            print(colorama.Fore.RED + f"Key not available for '{title} ( {creator_name} )'" + colorama.Style.RESET_ALL)
            else:
                skin_urls = []
                other_urls = []
                for item in result["items"]:
                    item_id = item.get("Id")
                    title = item.get("Title", {}).get("en-US", "")
                    creator_name = item.get("DisplayProperties", {}).get("creatorName", "")
                    for content in item.get("Contents", []):
                        if content.get("Type") == "skinbinary" or content.get("Type") == "personabinary":
                            skin_urls.append(content["Url"])
                        elif check_custom_id(item_id):
                            other_urls.append(content["Url"])
                        else:
                            print(colorama.Fore.RED + f"Key not available for '{title} ( {creator_name} )'" + colorama.Style.RESET_ALL)

        folders_to_remove = []

        for i, url in enumerate(other_urls):
            extracted_pack_folders = download_and_process_zip(url, output_folder)
            if extracted_pack_folders is None:
                continue
            addon_folders = []
            non_addon_folders = []
            for pack_name, pack_folder in extracted_pack_folders:
                is_addon_flag = check_for_addon(pack_folder)
                if is_addon_flag:
                    addon_folders.append(pack_folder)
                else:
                    non_addon_folders.append(pack_folder)
                folders_to_remove.append(pack_folder)

            if addon_folders:
                try:
                    first_uuid = data_uuid(pack_folder)
                    dlc.main(addon_folders, "keys.tsv", output_folder, is_addon=True)
                except Exception as e:
                    log_error(first_uuid, e)
            
            if non_addon_folders:
                try:
                    first_uuid = data_uuid(pack_folder)
                    dlc.main(non_addon_folders, "keys.tsv", output_folder, is_addon=False)
                except Exception as e:
                    log_error(first_uuid, e)
                        
        for folder in folders_to_remove:
            remove_extracted_folder(folder)


        for i, url in enumerate(skin_urls):
            extracted_pack_folders = download_and_process_zip(url, output_folder) 
            if extracted_pack_folders is None:
                continue
            for pack_name, pack_folder in extracted_pack_folders:
                try:
                    first_uuid = data_uuid(pack_folder)
                    skin.main(pack_folder, output_folder)
                    remove_extracted_folder(pack_folder)
                except Exception as e:
                    log_error(first_uuid, e)

if __name__ == "__main__":
    main()
