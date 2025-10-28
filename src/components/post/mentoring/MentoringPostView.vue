<template>
  <div id="mentoring-post-page">
    <HeaderView />
    <section class="community-banner">
      <h1>FASHION MENTORING</h1>
      <p>전문가와 함께 성장하세요</p>
    </section>

    <main class="main-container">
      <div class="post-column">
        <div v-if="isLoading" class="state">
          <p>데이터를 불러오는 중입니다...</p>
        </div>
        <div v-else-if="error" class="state error">
          <p>오류 발생: {{ error }}</p>
        </div>
        <article v-else-if="postData" class="post-card">
          <div class="post-header">
            <img :src="'/images/mentoringpost' + postId + '.jpg'" alt="작성자 프로필" class="avatar poster-avatar" @error="($event) => ($event.target.src = fallbackImage)" />
            <div class="user-info">
              <div class="user-name">
                <span>{{ postData.memberName || '작성자 정보 없음' }}</span>
              </div>
            </div>
            <div class="post-edit-actions" v-if="postData.author_num === currentMemberNum">
              <button @click="editPost">수정</button>
              <button @click="deletePost">삭제</button>
            </div>
          </div>

          <div class="post-body">
            <div class="tags">
              <span class="recruiting-badge" :class="{ closed: postData.FINISH === 1 }">{{ postData.FINISH === 0 ? '모집중' : '마감' }}</span>
            </div>
            <h2>{{ postData.title || '제목 없음' }}</h2>
            <img :src="'/images/mentoringpost' + postId + '.jpg'" alt="Mentoring default image" class="post-image" @error="($event) => ($event.target.src = '/images/defaultimage.png')" />
            <div class="post-content-text" v-html="postData.content || '내용 없음'"></div>
            <button class="report-button post-report-button" @click="reportPost(postId)">🚨 게시글 신고</button>
            <button v-if="postData.author_num === currentMemberNum" class="delete-button post-delete-button" @click="deletePost">🗑️ 게시글 삭제</button>
          </div>

          <div class="post-meta">
            <span>조회 {{ postData.views || 0 }}</span> <span>·</span>
            <span>댓글 {{ commentData?.length || 0 }}</span>
          </div>

          <section class="comment-section">
            <div class="comment-header">
              <h3>댓글 {{ commentData?.length || 0 }}</h3>
            </div>
            <ul class="comment-list" v-if="commentData && commentData.length > 0">
              <li v-for="comment in commentData" :key="comment.num" class="comment-item">
                <div class="avatar comment-avatar">{{ comment.memberName?.charAt(0) || '?' }}</div>
                <div class="comment-content">
                  <div class="comment-author-info">
                    <strong>{{ comment.memberName || 'Unknown User' }}</strong>
                  </div>
                  <p class="comment-text">{{ comment.content || '댓글 내용 없음' }}</p>
                </div>
                <div class="comment-edit-actions">
                  <button @click="reportComment(comment.num)">🚨 신고</button>
                  <button v-if="comment.member_num === currentMemberNum" @click="deleteComment(comment.num)">🗑️ 삭제</button>
                </div>
              </li>
            </ul>
            <p v-else>아직 댓글이 없습니다.</p>
            <form class="comment-form" @submit.prevent="handleCommentSubmit">
              <div class="avatar comment-avatar">{{ currentMemberName?.charAt(0) || '나'}}</div>
              <input type="text" placeholder="댓글을 입력해주세요" class="comment-input" v-model="newCommentText" />
              <button type="submit" class="comment-submit-button">등록</button>
            </form>
          </section>
        </article>
        <div v-else class="state">
          <p>게시글 데이터를 찾을 수 없습니다.</p>
        </div>
      </div>

      <aside class="sidebar-column">
         <div class="widget category-widget">
          <h3>카테고리</h3>
          <div class="category-list">
            <button v-if="categoriesLoading">로딩중...</button>
             <template v-else-if="categories.length > 0">
               <button v-for="category in categories" :key="category.num">
                 {{ category.NAME }}
               </button>
             </template>
             <button v-else disabled>카테고리 없음</button>
          </div>
        </div>
        <div class="widget mentors-widget">
          <h3><span class="icon">🏆</span> 인기 멘토</h3>
          <ul class="mentor-list">
             <li v-if="popularMentorsLoading">로딩 중...</li>
             <template v-else-if="popularMentors.length > 0">
                <li v-for="mentor in popularMentors" :key="mentor.num" @click="goToMentorPage(mentor.num)" :style="{ cursor: mentor.num ? 'pointer' : 'default' }">
                  <div class="mentor-info">
                    <strong>{{ mentor.name }}</strong>
                    <span>{{ mentor.field }}</span>
                  </div>
                  <div class="mentor-likes">
                    <span class="icon">⭐</span> {{ mentor.likes }}
                  </div>
                </li>
             </template>
             <li v-else>인기 멘토 없음</li>
          </ul>
        </div>
        <div class="widget cta-widget">
          <h3>멘토로 활동하기</h3>
          <p>패션 전문가와 함께하세요</p>
          <button class="cta-button" @click="goToApplyPage">신청하기</button>
        </div>
      </aside>
    </main>
    <FooterView/>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import axios from 'axios';
import HeaderView from '../../HeaderView.vue';
import FooterView from '../../FooterView.vue';

const route = useRoute();
const router = useRouter();

/* ================== axios 인스턴스 ================== */
const jsonServerApi = axios.create({
  baseURL: 'http://localhost:3000',
});
// Spring API용 (인증 필요 시) - 댓글 생성 시 사용
const api = axios.create({
  baseURL: '/api',
   withCredentials: true,
});
api.interceptors.request.use((config) => {
  const token = sessionStorage.getItem('token')
  if (token) {
    config.headers = config.headers || {}
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
});
api.interceptors.response.use(
  (res) => res,
  (err) => {
    if (err?.response?.status === 401) {
      alert('세션이 만료되었거나 권한이 없습니다. 다시 로그인해 주세요.')
      router.push('/') // 로그인 페이지로 리다이렉트
    }
    return Promise.reject(err)
  }
);


const postData = ref(null);
const commentData = ref([]);
const isLoading = ref(true);
const error = ref(null);

const newCommentText = ref('');
const postId = ref(null);
const fallbackImage = '/images/default_avatar.png';

// --- 현재 로그인 사용자 정보 ---
// 실제 구현 시 API 호출 또는 상태 관리 라이브러리 사용
const currentMemberNum = ref(4); // 예: user01 (이민준) - 본인 확인용
const currentMemberName = ref('이민준'); // 댓글 작성 시 표시될 이름
// ----------------------------

// --- 사이드바 데이터 ---
const categories = ref([]);
const categoriesLoading = ref(false);
const popularMentors = ref([]); // { num, name, field, likes }
const popularMentorsLoading = ref(false);
// -----------------------


onMounted(async () => {
  postId.value = route.params.id;
  if (!postId.value) {
    error.value = "게시글 ID가 주소에 포함되지 않았습니다.";
    isLoading.value = false;
    return;
  }
  await Promise.all([
      fetchPostAndComments(),
      fetchCategories(),
      fetchPopularMentors()
  ]);
});

// 카테고리(해시태그) 로딩 함수 (db.json Hash_Tag 테이블 사용, 상위 3개)
const fetchCategories = async () => {
  categoriesLoading.value = true;
  try {
    const response = await jsonServerApi.get('/Hash_Tag', {
        params: { _limit: 3 } // 상위 3개만 가져옴
    });
    categories.value = Array.isArray(response.data) ? response.data : [];
  } catch (e) {
    console.error('카테고리(해시태그) 조회 실패:', e);
    categories.value = [];
  } finally {
    categoriesLoading.value = false;
  }
};

// 인기 멘토 로딩 함수 (db.json Influencer_Page 테이블 num 2, 3, 4 사용)
const fetchPopularMentors = async () => {
   popularMentorsLoading.value = true;
   try {
     // Influencer_Page에서 num 2, 3, 4 인 데이터 조회
     const params = new URLSearchParams();
     params.append('num', 2);
     params.append('num', 3);
     params.append('num', 4);

     const response = await jsonServerApi.get(`/Influencer_Page?${params.toString()}`);
     const influencerPages = Array.isArray(response.data) ? response.data : [];

     // 각 Influencer_Page 데이터에 해당하는 Member 정보 조회 (good_count 가져오기 위해)
     const mentorDetails = await Promise.all(
       influencerPages.map(async (page) => {
         let memberInfo = {};
         try {
           const memberRes = await jsonServerApi.get(`/Member/${page.member_num}`);
           memberInfo = memberRes.data || {};
         } catch (memberErr) {
           console.warn(`Member ${page.member_num} 정보 조회 실패:`, memberErr);
         }
         return {
           num: page.member_num, // Member 번호 사용 (라우팅용)
           name: memberInfo.NAME || '알 수 없음',
           field: '전문 멘토', // 임시 필드
           likes: memberInfo.good_count || 0 // 좋아요 수
         };
       })
     );

     popularMentors.value = mentorDetails;

   } catch (e) {
     console.error('인기 멘토 조회 실패:', e);
     popularMentors.value = [];
   } finally {
     popularMentorsLoading.value = false;
   }
};

// 게시글 및 댓글 데이터 로딩 함수 (db.json 연동)
const fetchPostAndComments = async () => {
  isLoading.value = true;
  error.value = null;
  try {
    // 1. 게시글 정보 (json-server)
    const postResponse = await jsonServerApi.get(`/Mentoring_Post/${postId.value}`);
    let fetchedPost = postResponse.data;

    // 2. 작성자 정보 (json-server)
    if (fetchedPost && fetchedPost.author_num != null) {
      try {
        const memberResponse = await jsonServerApi.get(`/Member/${fetchedPost.author_num}`);
        fetchedPost.memberName = memberResponse.data?.NAME || '작성자 정보 없음';
      } catch (memberError) { fetchedPost.memberName = '정보 조회 실패'; }
    } else { fetchedPost.memberName = '작성자 정보 없음'; }
    postData.value = fetchedPost;


    // 3. 댓글 정보 (json-server)
    const commentsResponse = await jsonServerApi.get(`/Comment`, {
      params: { mentoring_post_num: postId.value }
    });
    let fetchedComments = Array.isArray(commentsResponse.data) ? commentsResponse.data : [];

    // 4. 댓글 작성자 정보 (json-server)
    if (fetchedComments.length > 0) {
       const commentAuthorNums = [...new Set(fetchedComments.map(c => c.member_num).filter(Boolean))];
       if (commentAuthorNums.length > 0) {
         const commentMemberParams = new URLSearchParams();
         commentAuthorNums.forEach(num => commentMemberParams.append('num', num));
          try {
             const commentMemberResponse = await jsonServerApi.get(`/Member?${commentMemberParams.toString()}`);
             const commentMemberMap = new Map((Array.isArray(commentMemberResponse.data) ? commentMemberResponse.data : []).map(m => [m.num, m.NAME]));
             fetchedComments = fetchedComments.map(c => ({ ...c, memberName: commentMemberMap.get(c.member_num) || '알 수 없음' }));
          } catch (commentMemberError) {
             fetchedComments = fetchedComments.map(c => ({ ...c, memberName: '정보 조회 실패' }));
          }
       } else {
          fetchedComments = fetchedComments.map(c => ({ ...c, memberName: '작성자 정보 없음' }));
       }
    }
    commentData.value = fetchedComments;

  } catch (err) {
    console.error("데이터 로딩 에러:", err);
    error.value = "게시글 정보를 불러오는 데 실패했습니다.";
    if (err.response?.status === 404) error.value = "해당 게시글을 찾을 수 없습니다.";
  } finally { isLoading.value = false; }
};

// 댓글 작성 (json-server API)
const handleCommentSubmit = async () => {
  if (!newCommentText.value.trim()) { alert("댓글 내용을 입력해주세요."); return; }
  if (!currentMemberNum.value) { alert("로그인이 필요합니다."); return; }

  try {
    const payload = {
        content: newCommentText.value.trim(),
        member_num: currentMemberNum.value,
        mentoring_post_num: Number(postId.value)
    };
    const response = await jsonServerApi.post(`/Comment`, payload);

    // 성공 시 새 댓글 정보 포함하여 로컬 목록 업데이트
    const newCommentData = {
        ...response.data,
        memberName: currentMemberName.value, // 현재 사용자 이름 사용
        userReaction: null,
        isReacting: false
    };
    commentData.value.push(newCommentData);
    newCommentText.value = '';

  } catch (err) { console.error("댓글 등록 에러:", err); alert("댓글 등록 실패"); }
};


// 게시글 수정 페이지 이동 (라우터 설정 필요)
const editPost = () => {
  alert('멘토링 게시글 수정 라우터 설정 필요');
  // router.push({ name: 'editMentoringPost', params: { id: postId.value } });
};

// 게시글 삭제 (json-server API)
const deletePost = async () => {
  if (!postData.value?.num) return;
  if (confirm('정말로 이 게시글을 삭제하시겠습니까?')) {
    try {
      await jsonServerApi.delete(`/Mentoring_Post/${postData.value.num}`);
      alert('게시글이 삭제되었습니다.');
      router.push({ name: 'mentoringboardview' }); // 게시판으로 이동
    } catch (err) { console.error("게시글 삭제 에러:", err); alert('게시글 삭제 실패'); }
  }
};

// 댓글 삭제 (json-server API)
const deleteComment = async (commentNum) => {
  if (!commentNum) { console.error("삭제할 댓글 번호가 없습니다."); return; }
  if (confirm('정말로 이 댓글을 삭제하시겠습니까?')) {
    try {
      await jsonServerApi.delete(`/Comment/${commentNum}`);
      alert('댓글이 삭제되었습니다.');
      commentData.value = commentData.value.filter(c => c.num !== commentNum);
    } catch (err) {
      console.error("댓글 삭제 에러:", err);
      const status = err.response?.status;
      if (status === 404) {
        alert(`댓글 삭제 실패: 댓글(ID: ${commentNum})을 찾을 수 없습니다.`);
      } else {
        alert(`댓글 삭제 실패 (오류: ${status || 'Unknown'})`);
      }
    }
  }
};

// 게시글 신고 페이지 이동
const reportPost = (postNum) => {
  if (!postNum) return;
  router.push({ name: 'reportMentoringPost', params: { num: postNum.toString() } });
};

// 댓글 신고 페이지 이동
const reportComment = (commentNum) => {
  if (!commentNum) return;
  router.push({ name: 'reportComment', params: { num: commentNum.toString() } });
};

// 인플루언서 페이지 이동 (라우터 이름 사용)
const goToMentorPage = (mentorNum) => {
  if (!mentorNum) return;
  // router/index.js 에 정의된 이름 사용: 'influencerpage-profile'
  router.push({ name: 'influencerpage-profile', params: { num: mentorNum.toString() } });
};

// 인플루언서 신청 페이지 이동
const goToApplyPage = () => {
  router.push({ name: 'influencerapply' });
};

</script>

<style scoped>
/* 이전 스타일 유지... */

/* 프로필 이미지 fallback 스타일 */
.avatar {
  background-color: #eee; /* 기본 배경색 */
  background-size: cover;
  background-position: center;
  color: #555; /* 글자 색 */
  display: flex; /* 내부 텍스트 중앙 정렬 */
  align-items: center;
  justify-content: center;
}
.avatar img { /* img 태그에 직접 적용 */
    width: 100%;
    height: 100%;
    object-fit: cover;
    border-radius: 50%; /* 이미지가 원형으로 보이도록 */
}


/* 게시글 신고/삭제 버튼 */
.report-button, .delete-button {
  display: inline-flex;
  padding: 4px 10px;
  font-size: 12px;
  border-radius: 6px;
  cursor: pointer;
  margin-top: 16px;
  font-weight: 500;
  margin-left: 8px; /* 버튼 간 간격 */
}
.report-button {
  background: #fff0f0;
  color: #d4183d;
  border: 1px solid #ffcccc;
}
.delete-button {
  background: #f3f4f6;
  color: #4b5563;
  border: 1px solid #e5e7eb;
}
.report-button:hover { background: #ffe0e0; }
.delete-button:hover { background: #e5e7eb; }


/* 댓글 신고/삭제 버튼 */
.comment-edit-actions button {
  font-size: 12px; /* 크기 살짝 줄임 */
  padding: 3px 6px;
}
.comment-edit-actions button:first-child { /* 신고 버튼 */
  color: #d4183d;
}
.comment-edit-actions button:first-child:hover {
  background-color: #fff0f0;
}
.comment-edit-actions button:last-child { /* 삭제 버튼 */
  color: #4b5563;
}
.comment-edit-actions button:last-child:hover {
  background-color: #f3f4f6;
}

/* 카테고리 로딩/없음 상태 */
.category-list button:disabled {
  cursor: not-allowed;
  opacity: 0.6;
  background-color: #e5e7eb;
  color: #6b7280;
}

/* 인기 멘토 로딩/없음 상태 */
.mentor-list li {
  cursor: default;
  transition: background-color 0.15s ease;
}
.mentor-list li[style*="cursor: pointer"]:hover {
  background-color: #f9fafb;
}

/* 나머지 스타일은 이전과 동일하게 유지 */
:root {
  --primary-color: #155DFC;
  --text-primary: #101828;
  --text-secondary: #364153;
  --text-light: #6A7282;
  --border-color: #E5E7EB;
  --bg-light: #F9FAFB;
  --bg-white: #FFFFFF;
  --separator-color: #E5E7EB;
  --recruiting-color: #008236;
  --recruiting-bg: #DCFCE7;
  --closed-color: #4B5563;
  --closed-bg: #F3F4F6;
}

#mentoring-post-page {
  font-family: 'ABeeZee', 'Arimo', sans-serif;
  background-color: var(--bg-white);
  color: var(--text-primary);
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

.community-banner {
  background: url('/images/FMbanner.jpg') center/cover no-repeat;
  color: white;
  text-align: center;
  padding: 3rem 1rem;
  position: relative;
}
.community-banner::before {
    content: "";
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: rgba(0, 0, 0, 0.3);
    z-index: 1;
}
.community-banner h1, .community-banner p {
    position: relative;
    z-index: 2;
}
.community-banner h1 {
  font-size: 36px;
  margin: 0 0 0.5rem 0;
}
.community-banner p {
  font-size: 16px;
  color: #D1D5DC;
  margin: 0;
}

.main-container {
  display: flex;
  justify-content: center;
  gap: 1.5rem;
  padding: 2rem;
  max-width: 1400px;
  margin: 0 auto;
  align-items: flex-start;
  flex: 1;
}

.post-column {
  flex: 2;
  max-width: 800px;
}
.post-card {
  background: var(--bg-white);
  border-radius: 10px;
  border: 1px solid var(--border-color);
  overflow: hidden;
  margin-bottom: 1.5rem;
}
.post-header {
  display: flex;
  align-items: center;
  padding: 1.5rem;
  gap: 0.75rem;
}
.avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  font-size: 16px;
  background: var(--border-color);
  color: var(--text-light);
  flex-shrink: 0;
  object-fit: cover;
}
.poster-avatar {
  background: var(--text-primary);
  color: white;
}
.user-info {
  flex: 1;
  text-align: left;
}
.user-name {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 14px;
  font-weight: bold;
}

.post-body {
  padding: 0 1.5rem 1.5rem;
  text-align: left;
}
.post-body h2 {
  font-size: 18px;
  color: var(--text-primary);
  margin: 0 0 0.75rem 0;
  font-weight: bold;
}
.tags {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-bottom: 1rem;
}
.tags span.recruiting-badge {
  font-size: 14px;
  cursor: default !important;
  padding: 4px 12px !important;
  border-radius: 4px;
  font-weight: bold !important;
  background-color: var(--recruiting-bg);
  color: var(--recruiting-color);
}
.tags span.recruiting-badge.closed {
  background-color: var(--closed-bg);
  color: var(--closed-color);
}

.post-image {
  width: 100%;
  height: auto;
  border-radius: 4px;
  margin-bottom: 1rem;
  object-fit: cover;
  max-height: 500px;
}
.post-content-text {
  font-size: 16px;
  color: var(--text-secondary);
  line-height: 1.7;
  white-space: pre-wrap;
}
.post-content-text p {
  margin: 0.5rem 0;
}
.post-content-text pre {
  background-color: var(--bg-light);
  padding: 1rem;
  border-radius: 4px;
  overflow-x: auto;
  font-family: monospace;
}

.post-meta {
  padding: 1rem 1.5rem;
  font-size: 14px;
  color: var(--text-light);
  text-align: left;
  border-top: 1px solid var(--separator-color);
}


.comment-section {
  padding: 1.5rem;
  border-top: 5px solid var(--separator-color);
}
.comment-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}
.comment-header h3 {
  font-size: 16px;
  margin: 0;
  font-weight: bold;
}

.comment-list {
  list-style: none;
  padding: 0;
  margin: 0;
}
.comment-item {
  display: flex;
  gap: 0.75rem;
  padding: 1rem 0;
  border-bottom: 1px solid #F3F4F6;
  align-items: flex-start;
  position: relative;
}
.comment-item:last-child {
  border-bottom: none;
}
.comment-avatar {
  width: 32px;
  height: 32px;
  font-size: 12px;
  margin-top: 4px;
}
.comment-content {
  flex: 1;
  text-align: left;
}
.comment-author-info {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin-bottom: 0.25rem;
}
.comment-author-info strong {
  font-size: 14px;
}
.comment-text {
  font-size: 14px;
  color: var(--text-secondary);
  margin: 0.5rem 0;
  word-break: break-word;
  white-space: pre-wrap;
}

.comment-form {
  display: flex;
  gap: 0.5rem;
  align-items: center;
  padding-top: 1rem;
  border-top: 1px solid #F3F4F6;
}
.comment-input {
  flex: 1;
  border: 1px solid var(--border-color);
  border-radius: 4px;
  padding: 0.75rem;
  font-size: 16px;
}
.comment-submit-button {
  padding: 0.75rem 1rem;
  background: var(--text-primary);
  color: var(--bg-white);
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.2s;
}
.comment-submit-button:hover {
    background-color: var(--text-secondary);
}

/* --- 사이드바 스타일 --- */
.sidebar-column {
  flex: 1;
  max-width: 390px;
  min-width: 300px;
}
.widget {
  background: var(--bg-white);
  border-radius: 10px;
  border: 1px solid var(--border-color);
  padding: 1.25rem;
  margin-bottom: 1.5rem;
  text-align: left;
}
.widget h3 {
  font-size: 16px;
  color: var(--text-primary);
  margin: 0 0 1rem 0;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-weight: bold;
}
.widget h3 .icon {
  font-size: 1.2em;
}

.category-list {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
}
.category-list button {
  padding: 0.5rem 0.75rem;
  border: none;
  border-radius: 4px;
  background: var(--bg-light);
  color: var(--text-secondary);
  font-size: 14px;
  cursor: pointer;
}
.category-list button.active {
  background: var(--text-primary);
  color: var(--bg-white);
}

.mentor-list {
  list-style: none;
  padding: 0;
  margin: 0;
}
.mentor-list li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.75rem 0;
  border-bottom: 1px solid #F3F4F6;
}
.mentor-list li:last-child {
  border-bottom: none;
}
.mentor-info {
  display: flex;
  flex-direction: column;
}
.mentor-info strong {
  font-size: 14px;
  color: var(--text-primary);
  font-weight: 600;
}
.mentor-info strong:hover {
  color: var(--primary-color);
  text-decoration: underline;
}
.mentor-info span {
  font-size: 12px;
  color: var(--text-light);
  margin-top: 2px;
}
.mentor-likes {
  font-size: 12px;
  color: #99A1AF;
  display: flex;
  align-items: center;
  gap: 2px;
  white-space: nowrap;
}

.cta-widget {
  background: linear-gradient(135deg, #155DFC 0%, #51A2FF 100%);
  color: white;
  text-align: center;
}
.cta-widget h3 {
  color: white;
  justify-content: center;
}
.cta-widget p {
  color: #DBEAFE;
  font-size: 14px;
  margin: 0.5rem 0 1rem 0;
}
.cta-button {
  width: 100%;
  padding: 0.75rem;
  background: var(--bg-white);
  color: var(--primary-color);
  border: none;
  border-radius: 4px;
  font-size: 14px;
  font-weight: bold;
  cursor: pointer;
  transition: background-color 0.2s, color 0.2s;
}
.cta-button:hover {
  background-color: #f0f5ff;
}

.state {
  text-align: center;
  color: var(--text-light);
  padding: 2rem;
}
.state.error {
  color: #e53935;
}

.post-edit-actions, .comment-edit-actions {
  display: flex;
  gap: 8px;
  margin-left: auto;
}
.post-edit-actions button, .comment-edit-actions button {
  background: none;
  border: none;
  color: var(--text-light);
  font-size: 13px;
  cursor: pointer;
  padding: 4px 8px;
  border-radius: 4px;
}
.post-edit-actions button:hover, .comment-edit-actions button:hover {
  background-color: var(--bg-light);
  color: var(--text-primary);
}
.comment-item {
  position: relative;
}
.comment-edit-actions {
  position: absolute;
  top: 1rem;
  right: 0;
}
</style>