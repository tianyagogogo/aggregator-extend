## Dockerfile
```
FROM python:alpine3.20
RUN apk update && apk add git
WORKDIR /app
COPY merged2upload.py /app/merged2upload.py
COPY entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh
ENV GITHUB_TOKEN=
ENV GITHUB_GIST_ID=
ENV PROXY=
ENV GIT_CLONE_PROXY=
ENTRYPOINT ["/app/entrypoint.sh"]
```

## entrypoint.sh
```
#!/bin/sh

# 启用错误检测
set -e

# 定义错误处理函数
error_handler() {
    echo "脚本在第 ${LINENO} 行发生错误"
    exit 1
}

# 捕获 ERR 信号，调用错误处理函数
trap 'error_handler' ERR

# clone 代码
# 检查环境变量是否存在且为1
if [ "${GIT_CLONE_PROXY}" = "1" ]; then
    echo "克隆 aggregator（使用 ghproxy 代理）"
    git clone --depth 1 https://mirror.ghproxy.com/https://github.com/wzdnzd/aggregator.git
else
    echo "克隆 aggregator"
    git clone --depth 1 https://github.com/wzdnzd/aggregator.git
fi

# 安装依赖
pip3 install pyYAML tqdm requests
# 设置代理
export https_proxy=$PROXY http_proxy=$PROXY all_proxy=$PROXY
# 运行代码
cd /app/aggregator && python -u subscribe/collect.py -si
python /app/merged2upload.py

```

## merged2upload.py
```
import requests
import base64
import os

def fetch_and_decode_base64(url):
    print(url)
    try:
        response = requests.get(url, verify=False)
        response.raise_for_status()
        decoded_content = base64.b64decode(response.text)
        return decoded_content.decode('utf-8')
    except requests.RequestException as e:
        print(f"Error fetching {url}: {e}")
        return None

def upload_to_gist(content, gist_id, github_token):
    url = f"https://api.github.com/gists/{gist_id}"
    headers = {
        'Authorization': f'token {github_token}',
        'Accept': 'application/vnd.github.v3+json'
    }
    data = {
        "files": {
            "configsub.yaml": {
                "content": content
            }
        }
    }
    try:
        response = requests.patch(url, headers=headers, json=data)
        response.raise_for_status()
        print(f"Successfully updated Gist: {gist_id}")
    except requests.RequestException as e:
        print(f"Error updating Gist: {e}")

def main():
    file_path = '/app/aggregator/data/subscribes.txt'

    with open(file_path, 'r') as file:
        urls = file.read().strip().split('\n')

    all_decoded_texts = []

    for url in urls:
        decoded_content = fetch_and_decode_base64(url)
        if decoded_content:
            all_decoded_texts.append(decoded_content)

    merged_content = "\n".join(all_decoded_texts)
    encoded_merged_content = base64.b64encode(merged_content.encode('utf-8')).decode('utf-8')

    merged_file_path = '/app/aggregator/data/merged.txt'
    with open(merged_file_path, 'w') as file:
        file.write(encoded_merged_content)
        print(f"Encoded merged content written to {merged_file_path}")

    # Upload the merged content to the Gist
    github_token = os.getenv('GITHUB_TOKEN')
    gist_id = os.getenv('GITHUB_GIST_ID')
    upload_to_gist(encoded_merged_content, gist_id, github_token)

if __name__ == "__main__":
    main()

```

