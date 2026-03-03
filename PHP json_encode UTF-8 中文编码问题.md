json_encode 使用 UTF-8 编码字符串，但默认会将中文转码为 Unicode 代码点的形式。这种编码方式确保了 JSON 字符串在任何编码环境下都能被正确解析，但会导致中文等字符在查看时不够直观。可以通过增加 JSON_UNESCAPED_UNICODE 参数来防止非 ASCII 字符被转义，而是直接以原始字符形式输出。
