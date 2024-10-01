import boto3
import logging
import os
from botocore.config import Config
import zipfile
import shutil
import datetime

# Setup logging
logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.INFO, format="%(levelname)s: %(message)s")

# Constants
MODEL_ID = 'anthropic.claude-3-5-sonnet-20240620-v1:0'
ACCESS_KEY_ID = ''
SECRET_ACCESS_KEY = ''
SESSION_TOKEN = ''
DIRECTORY_PATH = 'C:\GameRepos\game-adapter-6583-Release-1.0.0-1121'

def setup_aws_session(access_key_id, secret_access_key, session_token):
    """
    Setup the AWS session with the provided credentials.
    """
    boto3.setup_default_session(
        region_name='us-east-1',
        aws_access_key_id=access_key_id,
        aws_secret_access_key=secret_access_key,
        aws_session_token=session_token
    )

def get_bedrock_runtime_client():
    """
    Get the Bedrock runtime client with custom configuration.
    """
    config = Config(
        connect_timeout=60,
        read_timeout=600
    )
    return boto3.client(
        service_name='bedrock-runtime',
        region_name='us-east-1',
        config=config
    )

def get_comment_syntax(file_path):
    """
    Determine the appropriate comment syntax based on the file extension.
    """
    ext = os.path.splitext(file_path)[1].lower()
    if ext in ['.py']:
        return '#', ''
    elif ext in ['.java', '.css', '.ts', '.js']:
        return '//', ''
    elif ext in ['.html']:
        return '<!--', '-->'
    else:
        return None, None #Unknown file type

def list_files_with_prompts(directory):
    """
    List all relevant files in the given directory, excluding certain directories and file types,
    and return a list of tuples with the file path and the corresponding prompt.
    """
    files_with_prompts = []
    exclude_dirs = {'node_modules', 'venv', '__pycache__', 'public', 'test', 'tests', 'config'}
    exclude_files = {
        'package.json', 'package-lock.json', 'tsconfig.json', 'tslint.json',
        'angular.json', 'karma.conf.js', 'protractor.conf.js',
        '.gitignore', '.dockerignore', 'Dockerfile', 'docker-compose.yml',
        'README.md', 'LICENSE', 'CHANGELOG.md'
    }
    exclude_patterns = ['test', 'spec', 'mock', 'config', 'conf']

    
    prompt = """
    Below I have attached the contents of a file that contributes to an Application called GAME (GI Admin Management Engine).  This App consists of a bunch of UI (angular) and backend (Java) files that contribute to the final application.
    The app is mainly used by tech leads at a company so that they can get insights into different areas of their tech stack.  The app is like a one stop shop for many external tools like rally and github and it combines data from many places to benefit the end user.
    Please use this information to help guide your commenting of the code - reference the app at large when possible to give context to the code you are commenting.
    Task Instructions: 
    1) Please add code comments that explain the code's purpose and logic without modifying or changing the code itself. Your response should only include the original file with all of the added comments.
    2) Your comments should focus on both functionality (what the code does) as well as logic (how the code does it). 
    3) IMPORTANT - make sure that the comments are detailed and descriptive so that someone reading the code can understand the purpose of each part of the code.  When possible - take opportunities to explain how the code plays into the bigger picture of the application at large.  
    3 continued) If it interacts with other files or repositories related to the project, make sure to describe them and explain the connection.
    4) The file shoudnt be modified in any way we dont want anything added or removed from the file.
    5) Each class/file should have a multi-line comment right after any import statements that explains what the file does, what data and other files it interacts with, and what the file contains. These are files from repositories that are part of a larger project, so the imports are necessary for the code to run do not change them or modify them.
    6) Each function should include a function overview. This overview should have a high-level description of what the function does, it should have descriptions of the parameters, the data returned, and the exceptions it could throw.
    7) If a line of code is very standard or simple, you do not need to add a comment. Only for areas that are a bit more complex, or unclear from the code. For example - if there are variable or function names that are unclear - add a comment explaining what they are and what they do.
    7a) If there are any lines of code that are unclear or hard to understand, please add comments to explain them.
    8) If there are existing comments, do not overwrite them - instead add to them or explain them further. 
    9) Use the proper commenting syntax for the language in the file for both block comments and line by line comments as well.
    10) If the code is a UI Based component be sure to explain what it may look like and how the user is intended to interact with it.
Response Expectations and Formatting: 
1) If the file is too long to complete in one response, pause your answer and I will prompt you to continue. Once the entire file is completed, please write "DONE!" at the end.
2) Do not add an introduction or conclusion. 
3) Do not include (Here is the code with comments added:) or (Here is the commented code:) in your response.
4) Do not add any explanation at the end. 
5) We do not want any ```python or ```java at the start and end either, but remember both languages have different syntax so keep that in mind to use the proper syntax in each file accordingly. 
6) Do not abbreviate the code at all - we want the file exactly how it was except with comments added. Your task is to document the code with comments.
7) NEVER remove or modify the existing code - only add comments to it.
7B) There will be times will there will be big stretches of repetive code, like import statements or similar variable declarations.  When this happens, do not add one blanket comment and remove the code comment.  All of the source code should be part of your output.  If you remove source code, you did something wrong.
8) Even if the file is very big and your response won't fit in one response, don't worry.  Do as much as you can and do not compromise the quality of your comments.  We will prompt you to continue when you are ready.  Do not remove any of the code to fit in more comments.  If you remove code, you did something wrong.
CRITICAL: Do not remove, summarize, or abbreviate ANY part of the original code. The entire source code must be preserved exactly as it is, with comments added. This includes all import statements, method declarations, and any repetitive code. Removing or modifying any part of the source code is strictly prohibited.
Before you begin your response, read and understand the entire file, and identify what coding language is being used. When you are ready to begin your response - start immediately by outputting the first line of the file and add comments as necessary when you go forward."""

    for root, dirs, files in os.walk(directory):
        # Modify dirs in-place to exclude certain directories
        dirs[:] = [d for d in dirs if d.lower() not in exclude_dirs]
        
        for file in files:
            if file.lower() in exclude_files:
                continue

            if file.endswith(('.py', '.java', '.css', '.js', '.html', '.ts')):
                # Skip files based on patterns
                if any(pattern in file.lower() for pattern in exclude_patterns):
                    continue

                file_path = os.path.join(root, file)
                
                # Skip files in mock-data directories
                if 'mock-data' in file_path:
                    continue

                # Check if file has already been documented
                try:
                    with open(file_path, 'r') as f:
                        first_line = f.readline().strip()
                        if "Documented on" in first_line:
                            logger.info(f"Skipping already documented file: {file_path}")
                            continue
                except Exception as e:
                    logger.warning(f"Could not read file {file_path}: {e}")
                    continue

                files_with_prompts.append((file_path, prompt))

    return files_with_prompts


def process_large_file(brt, file_path, file_number, prompt):
    """
    Process a large file by sending its content to the Bedrock client for commenting.
    """
    file_name = os.path.basename(file_path)
    logger.info(f"Starting comment process for file {file_number + 1}: {file_name}")
    
    comment_start, comment_end = get_comment_syntax(file_path)
    
    # Read the entire file content
    with open(file_path, 'r') as file:
        file_content = file.read()

    initial_message = {
        "role": "user",
        "content": [{"text": f"{prompt}\n\n{file_content}"}]
    }

    messages = [initial_message]
    system_prompts = [{"text": "You are an app that adds comments to code files. Your response should only include the original file with all of the added comments. We shouldn't see the code differently, the code should remain the same, but with comments added."}]

    full_response = ""
    model_hit_count = 0
    previous_response_text = ""

    while True:
        logger.info(f"Hitting model with converse API for file {file_number + 1}: {file_name}")
        model_hit_count += 1
        response = brt.converse(
            modelId=MODEL_ID,
            messages=messages,
            system=system_prompts,
            inferenceConfig={"temperature": 0.7},
            additionalModelRequestFields={"top_k": 5}
        )

        output_message = response['output']['message']
        response_text = output_message['content'][0]['text'] if 'content' in output_message and output_message['content'] else ''

        # Ensure only comments are added, not extra content
        response_text = remove_code_block_markers(response_text)

        # Prevent duplication of content
        if response_text == previous_response_text:
            break

        full_response += response_text
        previous_response_text = response_text

        if "DONE!" in response_text:
            break

        # Add the assistant's response to the messages
        messages.append({
            "role": "assistant",
            "content": [{"text": response_text}]
        })

        # Prepare for the next iteration if needed
        messages.append({
            "role": "user",
            "content": [{"text": "Please continue the comments with the exact same task instructions and format as before.  Reminder: Do not modify the code, only add comments.  Do not add an introduction or conclusion.  Please pick up right where you left off.  When you are done simply write 'DONE!'"}]
        })

    logger.info(f"Commenting completed for file {file_number + 1}: {file_name}. Model hit count: {model_hit_count}")

    # Add documentation date to the top of the file
    today = datetime.date.today().strftime("%Y-%m-%d")
    if comment_end:  # For HTML files
        doc_line = f"{comment_start} Documented on {today} {comment_end}\n\n"
    else:
        doc_line = f"{comment_start} Documented on {today}\n\n"
    
    full_response_with_doc = doc_line + full_response.replace("DONE!", "").strip()

    # Save the commented content back to the file
    with open(file_path, 'w') as file:
        file.write(full_response_with_doc)

def comment_code(directory):
    """
    Process files in the given directory, add comments, and save the results.
    """
    try:
        logger.info("Starting the comment_code process")  # Log the start of the process

        # List all .py and .java files in the directory with their corresponding prompts
        py_java_files_with_prompts = list_files_with_prompts(directory)  # Get the list of Python and Java files with prompts
        total_files = len(py_java_files_with_prompts)  # Get the total number of files
        logger.info(f"Total number of files to process: {total_files}")  # Log the total number of files

        # Initialize AWS session and Bedrock client
        setup_aws_session(
            access_key_id=ACCESS_KEY_ID,  
            secret_access_key=SECRET_ACCESS_KEY,  
            session_token=SESSION_TOKEN  
        )
        bedrock_client = get_bedrock_runtime_client()  

        for idx, (file_path, prompt) in enumerate(py_java_files_with_prompts):  
            process_large_file(bedrock_client, file_path, idx, prompt)  

        # Get the original directory name
        original_dir_name = os.path.basename(os.path.normpath(directory))

        # Create a zip file of the entire directory, maintaining the directory structure
        zip_filename = f'{original_dir_name}.zip'  # Set the zip file name to the original directory name
        with zipfile.ZipFile(zip_filename, 'w') as zipf:  
            for root, _, files in os.walk(directory):  
                for file in files:  
                    file_path = os.path.join(root, file)  # Get the file path
                    arcname = os.path.relpath(file_path, start=os.path.dirname(directory))  # Get the relative file path including the original directory name
                    zipf.write(file_path, arcname)  


        logger.info(f"Created zip file: {zip_filename}")  


        # Move the zip file to the Downloads folder
        downloads_folder = os.path.join(os.path.expanduser("~"), "Downloads")  

        # Create the Downloads folder if it doesn't exist
        if not os.path.exists(downloads_folder):  
            os.makedirs(downloads_folder)  

        shutil.move(zip_filename, os.path.join(downloads_folder, zip_filename))  
        logger.info(f"Moved zip file to Downloads folder: {downloads_folder}")  

        logger.info("Completed the comment_code process")  #

    except Exception as e:
        logger.error(f"Failed to send file content to Claude: {e}")  

def remove_code_block_markers(response_text):
    """
    Remove code block markers for Python, Java, JavaScript, CSS, HTML, and generic code blocks from the response text.
    """
    markers = ["```python", "```java", "```js", "```css", "```html", "```"]
    for marker in markers:
        response_text = response_text.replace(marker, "")
    return response_text.strip()

if __name__ == '__main__':
    comment_code(DIRECTORY_PATH)  
