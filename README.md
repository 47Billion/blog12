# Setup

Install Hexo

```
sudo npm install hexo -g
```

Clone and build.

```
git clone https://github.com/47Billion/blog.git
cd blog
npm install
```

# Run local server

```
hexo server
```

Blog - [http://localhost:4000][6034aa1d]
[6034aa1d]: http://localhost:4000 "http://localhost:4000"
Admin - [http://localhost:4000/admin
][779be60d]
[779be60d]: http://localhost:4000/admin "http://localhost:4000/admin"

# New post

```
hexo new "Title Of The Post"
```
This will create these -

```
▶ tree source/_posts
source/_posts
|-- Title-Of-The-Post       // Dir for assets
|-- Title-Of-The-Post.md    // Content
```

## Insert image

- Copy the image to 'Title-Of-The-Post' folder. Let's say the name of the image is 'python.png'
- Wherever you need to insert this image, add this
```
{% asset_img python.png 'Image-Alt-Text' %}
```
