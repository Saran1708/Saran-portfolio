
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>AI Fashion Studio | Virtual Try-On</title>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css" rel="stylesheet">
  <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs"></script>
  <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/body-pix"></script>
  <style>
    .preview-container {
      border: 2px dashed #ddd;
      border-radius: 10px;
      min-height: 300px;
      background: #f8f9fa;
    }
    .result-image {
      transition: transform 0.3s ease;
      border: 3px solid #fff;
      box-shadow: 0 0.5rem 1rem rgba(0,0,0,0.15);
    }
    .result-image:hover {
      transform: scale(1.02);
    }
    .gradient-bg {
      background: linear-gradient(135deg, #f8f9fa 0%, #e9ecef 100%);
    }
    .loading-dots:after {
      content: '.';
      animation: dots 1.5s steps(5, end) infinite;
    }
    @keyframes dots {
      0%, 20% { content: '.'; }
      40% { content: '..'; }
      60% { content: '...'; }
      80%, 100% { content: ''; }
    }
  </style>
</head>
<body class="gradient-bg">

<div class="container py-5">
  <div class="text-center mb-5">
    <h1 class="display-4 fw-bold text-primary mb-3">OUTFIT AI</h1>
    <p class="lead text-muted">Try clothes virtually using AI-powered technology</p>
  </div>

  <div class="card shadow-lg">
    <div class="card-body p-4">
      <form id="uploadForm" enctype="multipart/form-data">
        <div class="row g-4">
          <div class="col-md-6">
            <div class="preview-container d-flex align-items-center justify-content-center p-3">
              <div id="imagePreview" class="text-center">
                <span class="text-muted">Upload preview will appear here</span>
              </div>
            </div>
          </div>
          
          <div class="col-md-6">
            <div class="mb-3">
              <label class="form-label fw-bold">Upload Your Photo</label><br>
              <input type="file" class="form-control" id="upload" name="image" accept="image/*" required>
            </div>

            <div class="mb-4"><br>
              <label class="form-label fw-bold">Style Description</label>
              <input type="text" class="form-control" name="prompt" 
                     placeholder="e.g. 'A modern red leather jacket with black trim'" required>
            </div>
<br>
            <button type="submit" class="btn btn-primary w-100 btn-lg">
              Generate Virtual Try-On
            </button>
          </div>
        </div>
      </form>
    </div>
  </div>

  <div id="loadingSection" class="mt-4 text-center d-none">
    <div class="spinner-border text-primary" role="status" style="width: 3rem; height: 3rem;">
      <span class="visually-hidden">Loading...</span>
    </div>
    <h4 class="mt-3 text-primary loading-dots">Processing your image</h4>
    <p class="text-muted mt-2" id="statusText">Initializing AI models</p>
  </div>

  <div id="result" class="row mt-5 g-4 d-none">
    <div class="col-md-6">
      <div class="card h-100 shadow">
        <div class="card-body">
          <h5 class="card-title">Original Photo</h5>
          <div id="originalImageContainer"></div>
        </div>
      </div>
    </div>
    <div class="col-md-6">
      <div class="card h-100 shadow">
        <div class="card-body">
          <h5 class="card-title">AI Generated Result</h5>
          <div id="generatedImageContainer"></div>
          <div class="mt-3 text-center" id="downloadSection"></div>
        </div>
      </div>
    </div>
  </div>
</div>

<script>
  const form = document.getElementById('uploadForm');
  const loadingSection = document.getElementById('loadingSection');
  const statusText = document.getElementById('statusText');
  const resultSection = document.getElementById('result');
  const imagePreview = document.getElementById('imagePreview');
  const originalImageContainer = document.getElementById('originalImageContainer');
  const generatedImageContainer = document.getElementById('generatedImageContainer');
  const downloadSection = document.getElementById('downloadSection');
  let net;

  // Initialize BodyPix
  bodyPix.load().then(loadedNet => {
    net = loadedNet;
    statusText.textContent = 'AI models ready';
  });

  // Handle image preview
  document.getElementById('upload').addEventListener('change', function(e) {
    const file = e.target.files[0];
    if (file) {
      const reader = new FileReader();
      reader.onload = function(e) {
        imagePreview.innerHTML = `
          <img src="${e.target.result}" 
               class="img-fluid rounded result-image" 
               style="max-height: 300px">`;
      }
      reader.readAsDataURL(file);
    }
  });

  form.addEventListener('submit', async (e) => {
    e.preventDefault();
    loadingSection.classList.remove('d-none');
    resultSection.classList.add('d-none');
    statusText.textContent = 'Analyzing body pose...';

    const file = document.getElementById('upload').files[0];
    const img = new Image();
    img.src = URL.createObjectURL(file);

    img.onload = async () => {
      statusText.textContent = 'Segmenting clothing areas...';
      const segmentation = await net.segmentPersonParts(img);

      const canvas = document.createElement('canvas');
      canvas.width = img.width;
      canvas.height = img.height;
      const ctx = canvas.getContext('2d');
      const imageData = ctx.createImageData(canvas.width, canvas.height);
      const shirtPartIds = [2, 3, 4, 5, 9, 10, 11, 12];
      
      for (let i = 0; i < segmentation.data.length; i++) {
        const partId = segmentation.data[i];
        const pixelIndex = i * 4;
        if (shirtPartIds.includes(partId)) {
          imageData.data.set([255, 255, 255, 255], pixelIndex);
        } else {
          imageData.data.set([0, 0, 0, 255], pixelIndex);
        }
      }

      ctx.putImageData(imageData, 0, 0);
      statusText.textContent = 'Generating new clothing...';

      canvas.toBlob(async (maskBlob) => {
        const formData = new FormData();
        formData.append('image', file);
        formData.append('mask', maskBlob, 'mask.png');
        formData.append('prompt', document.querySelector('input[name="prompt"]').value);

        try {
          const response = await fetch('/process', { method: 'POST', body: formData });
          if (!response.ok) throw new Error('Server error');
          const data = await response.blob();
          
          loadingSection.classList.add('d-none');
          resultSection.classList.remove('d-none');

          const originalUrl = URL.createObjectURL(file);
          const resultUrl = URL.createObjectURL(data);
          
          originalImageContainer.innerHTML = `
            <img src="${originalUrl}" class="img-fluid rounded result-image">`;
          
          generatedImageContainer.innerHTML = `
            <img src="${resultUrl}" class="img-fluid rounded result-image">`;
          
          downloadSection.innerHTML = `
            <a href="${resultUrl}" download="virtual-tryon-result.png" 
               class="btn btn-success btn-lg">
              Download Your Design
            </a>`;
        } catch (error) {
          console.error(error);
          loadingSection.classList.add('d-none');
          alert('Error processing image. Please try again.');
        }
      }, 'image/png');
    };
  });
</script>

</body>
</html>



from flask import Flask, render_template, request, send_file
import os
from PIL import Image
import torch
from diffusers import StableDiffusionInpaintPipeline

app = Flask(__name__)
UPLOAD_FOLDER = 'static/uploads'
MASK_FOLDER = 'static/masks'
OUTPUT_FOLDER = 'static/output'
os.makedirs(UPLOAD_FOLDER, exist_ok=True)
os.makedirs(MASK_FOLDER, exist_ok=True)
os.makedirs(OUTPUT_FOLDER, exist_ok=True)

pipe = StableDiffusionInpaintPipeline.from_pretrained("runwayml/stable-diffusion-inpainting").to("cpu")

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/process', methods=['POST'])
def process():
    image_file = request.files['image']
    mask_file = request.files['mask']
    prompt = request.form.get('prompt')

    image_path = os.path.join(UPLOAD_FOLDER, 'input.png')
    mask_path = os.path.join(MASK_FOLDER, 'mask.png')
    output_path = os.path.join(OUTPUT_FOLDER, 'output.png')

    image_file.save(image_path)
    mask_file.save(mask_path)

    init_image = Image.open(image_path).convert("RGB").resize((512, 512))
    mask_image = Image.open(mask_path).convert("RGB").resize((512, 512))

    result = pipe(prompt=prompt, image=init_image, mask_image=mask_image).images[0]
    result.save(output_path)

    return send_file(output_path, mimetype='image/png')

if __name__ == '__main__':
    app.run(debug=True)