<template>
  <div class="post-create-container">
    <div class="post-create-card">
      <h1 class="title">{{ isEditMode ? '게시물 수정' : '새 게시물 만들기' }}</h1>
      
      <div class="post-create-layout">
        <!-- 왼쪽: 이미지 업로드 영역 -->
        <div class="left-section">
          <div class="image-upload-area"
               @click="handleImageAreaClick"
               @dragover.prevent
               @drop.prevent="handleDrop"
               :class="{ 'disabled': isEditMode }">
            <input type="file" 
                   ref="fileInput" 
                   @change="handleImageUpload" 
                   multiple 
                   accept="image/*" 
                   class="hidden"
                   :disabled="isEditMode">
            
            <div v-if="imagePreviews.length === 0" class="upload-placeholder">
              <i class="fas fa-cloud-upload-alt"></i>
              <p>{{ isEditMode ? '수정 시 이미지 변경 불가' : '이미지를 선택하세요' }}</p>
              <span v-if="!isEditMode" class="upload-hint">클릭하거나 드래그하여 업로드</span>
            </div>
            
            <div v-else class="image-slider">
              <button v-if="currentImageIndex > 0"
                      @click.stop="prevImage" 
                      class="slider-button prev">
                <i class="fas fa-chevron-left"></i>
              </button>

              <div class="image-preview">
                <img :src="imagePreviews[currentImageIndex]" alt="Preview">
                <div v-if="imagePreviews.length > 1" class="image-counter">
                  {{ currentImageIndex + 1 }} / {{ imagePreviews.length }}
                </div>
              </div>

              <button v-if="currentImageIndex < imagePreviews.length - 1"
                      @click.stop="nextImage" 
                      class="slider-button next">
                <i class="fas fa-chevron-right"></i>
              </button>
            </div>
          </div>
        </div>

        <!-- 오른쪽: 컨텐츠 입력 영역 -->
        <div class="right-section">
          <!-- 내용 입력 -->
          <div class="input-group">
            <label>내용</label>
            <textarea 
              v-model="content" 
              placeholder="당신의 이야기를 들려주세요..."
              class="content-input"
            ></textarea>
          </div>

          <!-- 해시태그 입력 -->
          <div class="input-group">
            <label>해시태그</label>
            <div class="hashtag-input-wrapper">
              <input 
                v-model="hashtagInput"
                @keyup.enter="addHashtag"
                placeholder="태그 입력 후 Enter"
                class="hashtag-input"
              >
            </div>
            <div class="hashtag-list">
              <span v-for="(tag, index) in hashtags" 
                    :key="index" 
                    class="hashtag-item">
                #{{ tag }}
                <button @click="removeHashtag(index)" 
                        class="remove-tag-btn">×</button>
              </span>
            </div>
          </div>

          <!-- 공개 범위 선택 -->
          <div class="visibility-group">
            <div class="visibility-options">
              <button class="visibility-btn" 
                      :class="{ active: visibility === 'ALL' }"
                      @click="visibility = 'ALL'">
                <i class="fas fa-globe"></i> 모두에게
              </button>
              <button class="visibility-btn"
                      :class="{ active: visibility === 'FOLLOW' }"
                      @click="visibility = 'FOLLOW'">
                <i class="fas fa-user-friends"></i> 맞팔로워만
              </button>
              <button class="visibility-btn"
                      :class="{ active: visibility === 'ONLYME' }"
                      @click="visibility = 'ONLYME'">
                <i class="fas fa-lock"></i> 나만보기
              </button>
            </div>
          </div>

          <!-- 게시하기 버튼 -->
          <button @click="createPost" 
                  class="submit-btn"
                  :disabled="!content || imagePreviews.length === 0">
            게시하기
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios'

export default {
  name: 'PostCreate',
  data() {
    return {
      isEditMode: false,
      postId: null,
      content: '',
      hashtagInput: '',
      hashtags: [],
      images: [],
      imagePreviews: [],
      visibility: 'ALL',
      isSubmitting: false,
      currentImageIndex: 0,
      isDragging: false
    }
  },
  async created() {
    // URL에서 postId가 있으면 수정 모드
    const postId = this.$route.params.postId
    if (postId) {
      this.isEditMode = true
      this.postId = postId
      await this.fetchPostData()
    }
  },
  methods: {
    handleImageAreaClick() {
      if (!this.isEditMode) {
        this.triggerFileInput();
      }
    },
    handleDrop(event) {
      if (this.isEditMode) return;
      
      const files = Array.from(event.dataTransfer.files).filter(file => 
        file.type.startsWith('image/')
      );
      if (files.length > 0) {
        this.processImages(files);
      }
      this.isDragging = false;
    },
    handleImageUpload(event) {
      if (this.isEditMode) return;
      
      const files = Array.from(event.target.files);
      if (files.length > 0) {
        this.processImages(files);
      }
    },
    processImages(files) {
      this.imagePreviews = [] // 기존 미리보기 초기화
      this.images = [] //기존 이미지 초기화
      
      files.forEach(file => {
        this.images.push(file) //파일 추가

        const reader = new FileReader()
        reader.onload = (e) => {
          this.imagePreviews.push(e.target.result)
        }
        reader.readAsDataURL(file)
      })
      
      this.currentImageIndex = 0
    },
    prevImage() {
      if (this.currentImageIndex > 0) {
        this.currentImageIndex--
      }
    },
    nextImage() {
      if (this.currentImageIndex < this.imagePreviews.length - 1) {
        this.currentImageIndex++
      }
    },

    // sanitize 함수: #, 대괄호, 따옴표, 쉼표, 백슬래시, 공백 제거
  sanitizeTag(tag) {
    if (tag == null) return ''
    return String(tag).replace(/[#\x5B\x5D"',\\\s]/g, '').trim()
  },

    // 태그 추가
    addHashtag() {
      const raw = this.hashtagInput || ''
      const cleaned = this.sanitizeTag(raw.replace(/^#/, ''))
      if (cleaned && !this.hashtags.includes(cleaned)) {
        this.hashtags.push(cleaned)
      }
      this.hashtagInput = ''
    },

    removeHashtag(index) {
      this.hashtags.splice(index, 1)
    },

    async fetchPostData() {
      try {
        const response = await axios.get(
          `${process.env.VUE_APP_API_BASE_URL}/post/detail/${this.postId}`,
          {
            headers: {
              'Authorization': `Bearer ${localStorage.getItem('token')}`,
            }
          }
        )
        
        if (response.data.status_code === 200) {
          const postData = response.data.result
          // 기존 데이터로 폼 초기화
          this.content = postData.content
          // 받아온 해시태그도 sanitize 해서 저장
          this.hashtags = (postData.hashTag || []).map(t => this.sanitizeTag(String(t))).filter(Boolean)
          
          // 이미지 URL을 미리보기로 설정
          this.imagePreviews = postData.imageList || []
          this.visibility = postData.visibility || 'ALL'
        }
      } catch (error) {
        console.error('게시물 데이터 로드 실패:', error)
      }
    },

    async createPost() {
      if (this.isSubmitting) return
      
      try {
        this.isSubmitting = true

        const sanitizedTags = (this.hashtags || []).map(t => this.sanitizeTag(String(t))).filter(Boolean)

        if (this.isEditMode) {
          // 수정 요청 데이터 구성 (JSON 바디)
          const updateData = {
            postId: this.postId,
            content: this.content,
            hashTag: sanitizedTags,
            visibility: this.visibility
          }

          const response = await axios.post(
            `${process.env.VUE_APP_API_BASE_URL}/post/update/${this.postId}`,
            updateData,
            {
              headers: {
                'Authorization': `Bearer ${localStorage.getItem('token')}`,
                'Content-Type': 'application/json'
              }
            }
          )

          if (response.data.status_code === 200) {
            this.$router.push(`/post/detail/${this.postId}`)
          }
        } else {
          // 새 게시물 작성 로직 (multipart/form-data)
          const formData = new FormData()
          formData.append('content', this.content)
          formData.append('visibility', this.visibility)
          
          // 해시태그: 각 태그를 개별 필드로 추가 (Spring @ModelAttribute List<String>로 바인딩 가능)
          sanitizedTags.forEach(tag => {
            formData.append('hashTag', tag)
          })
          
          // 이미지 파일들
          if (this.images.length > 0) {
            this.images.forEach(image => {
              formData.append('imageFile', image)
            })
          }

          const response = await axios.post(
            `${process.env.VUE_APP_API_BASE_URL}/post/create`,
            formData,
            {
              headers: {
                'Authorization': `Bearer ${localStorage.getItem('token')}`,
                'Content-Type': 'multipart/form-data'
              }
            }
          )

          if (response.data.status_code === 200) {
            this.$router.push(`/post/detail/${response.data.result}`)
          }
        }
      } catch (error) {
        console.error(this.isEditMode ? '게시물 수정 실패:' : '게시물 작성 실패:', error)
      } finally {
        this.isSubmitting = false
      }
    },

    triggerFileInput() {
      this.$refs.fileInput.click()
    }
  },
  mounted() {
    // 드래그 이벤트 리스너 추가
    const imageSection = this.$el.querySelector('.image-upload-area')
    
    imageSection.addEventListener('dragenter', (e) => {
      e.preventDefault()
      this.isDragging = true
    })
    
    imageSection.addEventListener('dragleave', (e) => {
      e.preventDefault()
      const rect = imageSection.getBoundingClientRect()
      if (
        e.clientX < rect.left ||
        e.clientX >= rect.right ||
        e.clientY < rect.top ||
        e.clientY >= rect.bottom
      ) {
        this.isDragging = false
      }
    })
  }
}
</script>

<style scoped>
.post-create-container {
  width: calc(100% - 240px);
  min-height: 100vh;
  margin-left: 240px;
  padding: 20px;
  background-color: #f8f9fa;
  display: flex;
  justify-content: center;
}

.post-create-card {
  position: relative;
  min-width: 240px;
  background: white;
  border-radius: 10px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  width: 100%;
  max-width: 1000px;
  padding: 20px;
}

.title {
  text-align: center;
  font-size: 30px;
  font-weight: 600;
  margin-bottom: 45px;
  margin-top: 30px;
  color: #333;
}

.post-create-layout {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;
}

.left-section {
  border: 2px dashed #e1e3e8;
  border-radius: 10px;
  min-height: 400px;
  position: relative;
}

.image-upload-area {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
}

.upload-placeholder {
  text-align: center;
  color: #666;
}

.upload-placeholder i {
  font-size: 48px;
  color: #4a90e2;
  margin-bottom: 15px;
}

.right-section {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.input-group {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.input-group label {
  font-weight: 600;
  color: #666;
}

.content-input {
  width: 100%;
  min-height: 150px;
  padding: 15px;
  border: 1px solid #e1e3e8;
  border-radius: 8px;
  resize: vertical;
}

.hashtag-input {
  width: 100%;
  padding: 10px;
  border: 1px solid #e1e3e8;
  border-radius: 8px;
}

.hashtag-list {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-top: 10px;
}

.hashtag-item {
  background: #f0f2f5;
  padding: 5px 10px;
  border-radius: 15px;
  font-size: 14px;
  display: flex;
  align-items: center;
  gap: 5px;
}

.visibility-options {
  display: flex;
  gap: 10px;
}

.visibility-btn {
  flex: 1;
  padding: 10px;
  border: 1px solid #e1e3e8;
  border-radius: 8px;
  background: white;
  cursor: pointer;
}

.visibility-btn.active {
  background: #4a90e2;
  color: white;
  border-color: #4a90e2;
}

.submit-btn {
  margin-top: auto;
  padding: 15px;
  background: #4a90e2;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-weight: 600;
}

.submit-btn:disabled {
  background: #b2dffc;
  cursor: not-allowed;
}

.hidden {
  display: none;
}

.image-slider {
  position: relative;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
}

.image-preview {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
}

.image-preview img {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
}

.slider-button {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  background: rgba(255, 255, 255, 0.9);
  border: none;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  z-index: 2;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
}

.slider-button:hover {
  background: white;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.slider-button.prev {
  left: 10px;
}

.slider-button.next {
  right: 10px;
}

.image-counter {
  position: absolute;
  top: 10px;
  right: 10px;
  background: rgba(0, 0, 0, 0.6);
  color: white;
  padding: 4px 8px;
  border-radius: 12px;
  font-size: 12px;
}

.image-preview img {
  max-width: 100%;
  max-height: 400px;
  object-fit: contain;
  transition: all 0.3s ease;
}

/* 이미지 전환 애니메이션 */
.image-preview {
  transition: transform 0.3s ease;
}

.upload-placeholder {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
  width: 100%;
}

/* 태블릿 크기 */
@media screen and (max-width: 1024px) {
  .post-create-container {
    width: 100%;
    margin-left: 240px;
    padding: 15px;
  }
  
  .post-create-layout {
    grid-template-columns: 1fr;
    gap: 15px;
  }
}

/* 모바일 크기 */
@media screen and (max-width: 768px) {
  .post-create-container {
    width: 100%;
    margin-left: 0;
    padding: 10px;
  }
  
  .post-create-card {
    min-width: 100%;
  }
  
  .title {
    font-size: 24px;
    margin: 20px 0;
  }
}

.image-upload-area.disabled {
  cursor: not-allowed;
  opacity: 0.7;
}

.image-upload-area.disabled:hover {
  background-color: #f8f9fa;
}
</style>