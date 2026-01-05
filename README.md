# mp3tag
mp3_editor/
│
├── app.py              # The backend logic
├── uploads/            # Folder to temporarily store files
└── templates/
    └── index.html      # The frontend user interface
    import os
from flask import Flask, render_template, request, send_file, redirect, url_for
from mutagen.easyid3 import EasyID3
from mutagen.mp3 import MP3
from werkzeug.utils import secure_filename

app = Flask(__name__)

# Configuration
UPLOAD_FOLDER = 'uploads'
os.makedirs(UPLOAD_FOLDER, exist_ok=True)
app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER

def get_tags(filepath):
    """Extracts tags from an MP3 file."""
    try:
        audio = MP3(filepath, ID3=EasyID3)
    except Exception:
        # If file has no tags, add a blank ID3 tag
        try:
            audio = MP3(filepath)
            audio.add_tags()
        except:
            return {}
    
    # Return a clean dictionary of tags
    return {
        'title': audio.get('title', [''])[0],
        'artist': audio.get('artist', [''])[0],
        'album': audio.get('album', [''])[0],
        'genre': audio.get('genre', [''])[0],
        'year': audio.get('date', [''])[0]
    }

@app.route('/', methods=['GET', 'POST'])
def index():
    if request.method == 'POST':
        # Check if a file was uploaded
        if 'file' not in request.files:
            return redirect(request.url)
        
        file = request.files['file']
        
        if file.filename == '':
            return redirect(request.url)

        if file:
            filename = secure_filename(file.filename)
            filepath = os.path.join(app.config['UPLOAD_FOLDER'], filename)
            file.save(filepath)
            
            # Read existing tags to pre-fill the form
            tags = get_tags(filepath)
            
            return render_template('index.html', tags=tags, filename=filename)

    return render_template('index.html', tags=None)

@app.route('/save', methods=['POST'])
def save_tags():
    filename = request.form['filename']
    filepath = os.path.join(app.config['UPLOAD_FOLDER'], filename)
    
    if not os.path.exists(filepath):
        return "File not found", 404

    # Load the file
    try:
        audio = MP3(filepath, ID3=EasyID3)
    except:
        audio = MP3(filepath)
        audio.add_tags()
    
    # Update tags from the form data
    audio['title'] = request.form.get('title')
    audio['artist'] = request.form.get('artist')
    audio['album'] = request.form.get('album')
    audio['genre'] = request.form.get('genre')
    audio['date'] = request.form.get('year')
    
    audio.save()
    
    # Send the modified file back to the user
    return send_file(filepath, as_attachment=True, download_name=f"edited_{filename}")

if __name__ == '__main__':
    app.run(debug=True)
    