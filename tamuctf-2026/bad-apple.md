# bad-apple
_Web Exploitation_

Credit: `moveslow`

## Description

funny touhou reference

[https://bad-apple.tamuctf.com](https://bad-apple.tamuctf.com)

## Downloads

[bad-apple](https://github.com/jasperjjhe/Capture-the-Flag/tree/main/tamuctf-2026/downloads)

## Approach

First thing I did was just try running the app and poking around. It plays the Bad Apple!! animation in ASCII-art style using extracted GIF frames...cute, but not immediately useful. Time to read the source.

The Dockerfile immediately gave away the flag's location:
```
CMD ["sh", "-c", "HEX=$(openssl rand -hex 16) && mv /srv/http/uploads/admin/flag.gif /srv/http/uploads/admin/$HEX-flag.gif ..."]
```
So the flag is a GIF sitting in `/srv/http/uploads/admin/`, but with a randomized 16-byte hex prefix we can't guess. We need to find the filename before we can do anything with it.

Looking at the Apache config:

```
    Alias /browse /srv/http/uploads
    <Directory /srv/http/uploads>
        Options +Indexes
        DirectoryIndex disabled
        IndexOptions FancyIndexing FoldersFirst NameWidth=* DescriptionWidth=* ShowForbidden
        AllowOverride None
        Require all granted

        <FilesMatch "\.gif$">
            AuthType Basic
            AuthName "Admin Area"
            AuthUserFile /srv/http/.htpasswd
            Require valid-user
        </FilesMatch>
    </Directory>
```

`+Indexes` enables directory listing with no auth. The auth gate only applies to actually downloading `.gif` files via `<FilesMatch>`. Browsing the directory is a different HTTP operation entirely, so visiting `/browse/admin/` will hand us the full filename for free.

Now we have the filename but can't download the `.gif` directly without credentials. The obvious next thought was: does the app ever touch these files server-side without going through Apache's auth layer? Looking at the `/convert` endpoint:

```
@app.route('/convert')
def convert():
    user_id = request.args.get('user_id', 'anonymous')
    filename = request.args.get('filename', '')

    input_path = os.path.join(app.config['UPLOAD_FOLDER'], secure_filename(user_id), filename)
    if not os.path.exists(input_path):
        return "File not found", 404

    safe_name = os.path.splitext(os.path.basename(filename))[0]
    output_dir = os.path.join(FRAMES_BASE, user_id, safe_name)
    os.makedirs(output_dir, exist_ok=True)

    try:
        frame_count = extract_frames(input_path, output_dir, safe_name)
        return redirect(url_for('index', view=safe_name, user_id=user_id))
    except Exception as e:
        return f"Error processing file: {str(e)}", 500
```

It just builds a filesystem path and reads it directly with no auth check whatsoever. Apache's `.htpasswd` only protects HTTP-level requests; the Flask process running as `apache` can read anything on the filesystem it has OS permissions for. So we hit:

```
/convert?user_id=admin&filename=<hex>-flag.gif
```
The server happily read the protected GIF and ran it through ffmpeg. Now we just needed to retrieve the output.

`extract_frames` saves everything to `/srv/http/static/frames/<user_id>/<gif_name>/`, which is served as a public static directory. It saves the individual PNG frames **and** a reconstructed `output.gif` midway through processing:
```
    cmd = [
        'ffmpeg', '-i', input_path,
        '-i', f'{output_dir}/palette.png',
        '-lavfi', f'fps=10,scale={width}:-1:flags=lanczos[x];[x][1:v]paletteuse',
        '-y', f'{output_dir}/output.gif'
    ]
```

So we visit:

```
/static/frames/admin/<hex>-flag/output.gif
```

🚩

## Summary

The challenge hid the flag behind two layers: a randomized filename and HTTP Basic Auth on the `.gif` file. Both layers had independent weaknesses that chained together.

Directory listing (`+Indexes`) on the uploads folder leaked the randomized filename without requiring any credentials since `<FilesMatch>` only gated file downloads, not directory browsing. Once we had the filename, the `/convert` endpoint let the server read and process the file at the OS level, bypassing Apache's auth. The processed output landed in a publicly accessible static directory and since ffmpeg had reconstructed the full `.gif` as `output.gif`, we could view it directly rather than scrubbing through individual frames.
