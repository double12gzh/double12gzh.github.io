---
layout: fragment
title: shell retry function
tags: [shell, function, retry]
description: demo on shell retry function
keywords: shell, retry, function
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

```bash
#!/bin/bash

retry_function() {
        local max_attempts=3
        local attempt=1

        while [ $attempt -le $max_attempts ]; do
                echo "Attempt $attempt:"

                # 接收传入的函数，"$@" 表示传递给函数的所有参数
                "$@"

                if [ $? -eq 0 ]; then
                        echo "Command succeeded!"
                        break
                else
                        echo "Command failed."
                        if [ $attempt -lt $max_attempts ]; then
                                echo "Retrying..."
                        else
                                echo "Max attempts reached. Exiting."
                                exit 1
                        fi
                fi

                ((attempt++))
                sleep 1 # 可选：在重试之间添加延迟
        done
}

# 传入的函数作为参数
example_function() {
        # 放入要重试的实际命令
        echo "Running the example function..."
        # 示例： ls -l
}

# 调用重试函数，并传入函数作为参数
retry_function example_function

```
