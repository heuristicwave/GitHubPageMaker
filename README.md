# 블로그 구조 분석 (Heuristic Wave Blog)

## 현재 블로그 아키텍처

### 기술 스택
- **정적 사이트 생성기**: Jekyll 3.6.2
- **호스팅**: GitHub Pages
- **테마**: Jasper2 (Ghost 테마 포팅) + **커스텀 다크 모드**
- **빌드 도구**: Gulp 4.0.2
- **CSS 전처리**: PostCSS + **CSS 커스텀 프로퍼티 시스템**
- **검색**: Lunr.js
- **분석**: Google Analytics (G-0FTXSPJZFY)
- **댓글**: Disqus
- **테마 시스템**: **라이트/다크 모드 지원**

### 디렉토리 구조

```
├── _config.yml              # Jekyll 설정
├── _data/                   # 데이터 파일
│   ├── authors.yml          # 작성자 정보
│   └── tags.yml             # 태그 정보
├── _includes/               # 재사용 가능한 템플릿 조각
│   ├── head.html            # HTML head 섹션
│   ├── navigation.html      # 네비게이션 메뉴
│   ├── post-card.html       # 포스트 카드 컴포넌트
│   ├── search-form.html     # 검색 폼
│   └── [기타 include 파일들]
├── _layouts/                # 페이지 레이아웃
│   ├── default.html         # 기본 레이아웃 + **다크 모드 시스템**
│   ├── post.html            # 포스트 레이아웃
│   ├── page.html            # 페이지 레이아웃
│   ├── author.html          # 작성자 페이지
│   └── tag.html             # 태그 페이지
├── _plugins/                # Jekyll 플러그인
│   ├── jekyll-autgenerator.rb
│   ├── jekyll-capitalize-all.rb
│   └── jekyll-tagsgenerator.rb
├── _posts/                  # 블로그 포스트
│   ├── ai/                  # AI 관련 포스트
│   ├── aws/                 # AWS 관련 포스트
│   ├── backend/             # 백엔드 관련 포스트
│   ├── devops/              # DevOps 관련 포스트
│   ├── security/            # 보안 관련 포스트
│   └── uncategorized/       # 미분류 포스트
├── assets/                  # 정적 자산
│   ├── built/               # 빌드된 CSS/JS
│   ├── css/                 # 소스 CSS + **커스텀 프로퍼티 시스템**
│   ├── js/                  # JavaScript 파일 + **테마 토글**
│   └── images/              # 이미지 파일
├── about/                   # About 페이지
├── package.json             # Node.js 의존성
├── Gemfile                  # Ruby 의존성
├── gulpfile.js              # Gulp 빌드 설정
└── buildspec.yaml           # AWS CodeBuild 설정
```

### 현재 기능

#### 핵심 기능
1. **포스트 관리**: 카테고리별 포스트 분류 (AI, AWS, Backend, DevOps, Security)
2. **검색**: Lunr.js 기반 클라이언트 사이드 검색
3. **페이지네이션**: 무한 스크롤 지원
4. **반응형 디자인**: 모바일 친화적 레이아웃
5. **SEO 최적화**: 메타 태그, 구조화된 데이터, 사이트맵
6. **소셜 미디어 통합**: Open Graph, Twitter Cards
7. **🆕 다크/라이트 모드**: 시스템 설정 감지 + 수동 전환
8. **🆕 스마트 테마 토글**: 네비게이션 통합 + 스크롤 시 플로팅

#### 스타일링
- **CSS 아키텍처**: PostCSS + Gulp 빌드 파이프라인
- **🆕 CSS 커스텀 프로퍼티**: 체계적인 디자인 토큰 시스템
- **🆕 테마 시스템**: 라이트/다크 모드 완전 지원
- **폰트**: Nanum Gothic (한글), Font Awesome (아이콘)
- **색상 스키마**: 
  - **라이트**: 밝은 배경 (#f4f8fb) 기반
  - **🆕 다크**: 어두운 배경 (#15171a) 기반, 네비게이션과 일치
- **코드 하이라이팅**: Highlight.js + Prism.js + **다크 모드 지원**

#### 🆕 테마 시스템 상세
- **자동 감지**: `prefers-color-scheme` 미디어 쿼리 지원
- **수동 전환**: 테마 토글 버튼으로 언제든 변경 가능
- **상태 저장**: localStorage에 사용자 선택 저장
- **스마트 UI**: 
  - 기본: 네비게이션 바에 자연스럽게 위치
  - 포스트 페이지 스크롤 시: 플로팅 버튼으로 전환
- **포괄적 지원**: 
  - 모든 텍스트 요소 (제목, 본문, 메타 정보)
  - 코드 블록 (인라인 코드, 코드 블록)
  - 강조 요소 (Bold, Italic, 인용구)
  - 테이블, 리스트, 링크
  - 카드 컴포넌트, 그림자, 테두리

#### 성능 최적화
- **이미지 최적화**: Gulp imagemin
- **CSS 최적화**: Autoprefixer, CSSnano
- **CDN 사용**: jQuery, Font Awesome, Highlight.js
- **🆕 테마 최적화**: 인라인 CSS로 깜빡임 방지

### 현재 아키텍처의 강점

1. **잘 구조화된 콘텐츠**: 카테고리별 명확한 분류
2. **SEO 친화적**: 메타 태그, 구조화된 데이터 완비
3. **성능 최적화**: 빌드 파이프라인을 통한 자산 최적화
4. **검색 기능**: 클라이언트 사이드 검색 구현
5. **반응형 디자인**: 모바일 친화적 레이아웃
6. **확장성**: 플러그인 시스템을 통한 기능 확장
7. **🆕 현대적 UX**: 다크 모드 지원으로 사용자 경험 향상
8. **🆕 접근성 개선**: 포커스 상태, 고대비 모드, 모션 감소 지원
9. **🆕 일관된 디자인**: CSS 커스텀 프로퍼티로 체계적인 디자인 시스템

### 🆕 최근 개선사항 (2025.09)

#### 테마 시스템 구현
- **CSS 커스텀 프로퍼티 시스템**: 색상, 간격, 타이포그래피 체계화
- **다크 모드 완전 지원**: 모든 UI 요소에 다크 테마 적용
- **스마트 테마 토글**: 상황에 맞는 UI 전환 (네비게이션 ↔ 플로팅)

#### UX/UI 개선
- **시각적 일관성**: 네비게이션 바와 일치하는 다크 테마 색상
- **텍스트 가독성**: Bold, 코드 블록 등 강조 요소 식별력 향상
- **인터랙션 개선**: 호버 효과, 전환 애니메이션 추가
- **반응형 최적화**: 모바일/태블릿에서 테마 토글 위치 조정

### 현재 아키텍처의 약점

1. **구식 기술 스택**: Jekyll 3.6.2, jQuery 3.2.1 등 오래된 버전
2. **복잡한 빌드 프로세스**: Gulp + Jekyll 이중 빌드
3. **제한적인 인터랙티브 기능**: 정적 사이트의 한계
4. **🔄 외부 서비스 의존**: Disqus 댓글 시스템의 테마 제어 한계

### 🔄 알려진 제한사항

#### 댓글 시스템 (Disqus)
- **외부 iframe**: Disqus는 독립적인 iframe으로 로드되어 부모 사이트의 CSS 제어 불가
- **테마 불일치**: 다크 모드에서 Disqus 댓글창이 여전히 밝은 테마로 표시
- **해결 방안**: 
  - Disqus 설정에서 다크 테마 활성화 (관리자 설정 필요)
  - 또는 대안 댓글 시스템 고려 (utterances, giscus 등)

## 🛠️ 개발 워크플로우

### 필수 명령어

#### CSS 빌드
```bash
# CSS 파일 빌드 (PostCSS 처리)
npx gulp css

# 이미지 최적화
npx gulp images

# 모든 자산 빌드
npx gulp
```

#### Jekyll 빌드
```bash
# 개발 서버 실행 (실시간 리로드)
bundle exec jekyll serve

# 개발 서버 (드래프트 포함)
bundle exec jekyll serve --drafts

# 프로덕션 빌드
bundle exec jekyll clean && bundle exec jekyll build

# 빌드 + 환경 변수 설정
JEKYLL_ENV=production bundle exec jekyll build
```

#### 통합 개발 워크플로우
```bash
# 1. CSS 변경 후 빌드
npx gulp css

# 2. Jekyll 재빌드
bundle exec jekyll clean && bundle exec jekyll build

# 3. 개발 서버 재시작
bundle exec jekyll serve
```

### 언제 어떤 명령어를 사용할까?

#### `npx gulp css`를 사용하는 경우
- ✅ `assets/css/` 폴더의 CSS 파일을 수정했을 때
- ✅ PostCSS 플러그인 설정을 변경했을 때
- ✅ CSS 커스텀 프로퍼티를 추가/수정했을 때
- ✅ 새로운 CSS 파일을 추가했을 때 (예: `search.css`, `toc.css`)
- ✅ Autoprefixer, CSSnano 등 PostCSS 처리가 필요할 때

#### `bundle exec jekyll clean && bundle exec jekyll build`를 사용하는 경우
- ✅ `_config.yml` 설정을 변경했을 때
- ✅ `_layouts/`, `_includes/` 템플릿을 수정했을 때
- ✅ 새로운 포스트를 추가했을 때
- ✅ `_data/` 폴더의 데이터 파일을 수정했을 때
- ✅ 플러그인을 추가/수정했을 때
- ✅ 빌드 오류가 발생했을 때 (clean으로 캐시 초기화)

#### `bundle exec jekyll serve`를 사용하는 경우
- ✅ 개발 중 실시간으로 변경사항을 확인하고 싶을 때
- ✅ 로컬에서 사이트를 미리보기할 때
- ✅ 포스트 작성 중 실시간 프리뷰가 필요할 때

### 개발 시나리오별 가이드

#### 🎨 CSS 스타일 수정
```bash
# 1. CSS 파일 수정 (예: assets/css/custom.css)
# 2. CSS 빌드
npx gulp css
# 3. 변경사항 확인 (Jekyll 서버가 실행 중이면 자동 리로드)
```

#### 📝 새 포스트 작성
```bash
# 1. _posts/ 폴더에 새 마크다운 파일 생성
# 2. Jekyll 서버 실행 (실시간 프리뷰)
bundle exec jekyll serve --drafts
# 3. 브라우저에서 http://localhost:4000 확인
```

#### 🔧 템플릿/설정 수정
```bash
# 1. _layouts/, _includes/, _config.yml 수정
# 2. 클린 빌드 (캐시 초기화)
bundle exec jekyll clean && bundle exec jekyll build
# 3. 서버 재시작
bundle exec jekyll serve
```

#### 🚀 배포 전 최종 빌드
```bash
# 1. CSS 최종 빌드
npx gulp css
# 2. 프로덕션 빌드
JEKYLL_ENV=production bundle exec jekyll clean && bundle exec jekyll build
# 3. _site/ 폴더 확인 후 배포
```

### 트러블슈팅

#### PostCSS 오류 발생 시
```bash
# 1. node_modules 재설치
rm -rf node_modules package-lock.json
npm install

# 2. CSS 빌드 재시도
npx gulp css
```

#### Jekyll 빌드 오류 시
```bash
# 1. 번들 재설치
bundle install

# 2. 클린 빌드
bundle exec jekyll clean
bundle exec jekyll build --verbose
```

#### 캐시 문제 해결
```bash
# Jekyll 캐시 초기화
bundle exec jekyll clean

# Gulp 캐시 초기화 (필요시)
rm -rf assets/built/
npx gulp css
```

### 성능 최적화 팁

#### 개발 중
- `jekyll serve`는 변경사항을 자동 감지하므로 CSS 수정 후 `npx gulp css`만 실행
- 대용량 이미지 추가 시 `npx gulp images`로 최적화

#### 배포 전
- 항상 `JEKYLL_ENV=production` 환경변수 설정
- CSS 압축 확인 (`assets/built/` 폴더)
- 이미지 최적화 확인

## 기술적 부채

### 의존성 관리
- 오래된 Node.js 패키지들
- 보안 취약점 가능성
- 호환성 문제 잠재

### 코드 품질
- ~~인라인 스타일 사용~~ → **🆕 체계적인 CSS 커스텀 프로퍼티로 개선**
- ~~중복된 CSS 규칙~~ → **🆕 통합된 테마 시스템으로 정리**
- 일관성 없는 네이밍 컨벤션 (일부 개선됨)

### 성능 이슈
- 여러 CSS/JS 라이브러리 중복 로드
- 이미지 최적화 개선 여지
- 번들 크기 최적화 필요

## 🎯 향후 개선 계획

### 단기 (1-2개월)
1. **Disqus 다크 모드 설정** 또는 대안 댓글 시스템 도입
2. **Jekyll 버전 업그레이드** (보안 및 성능 개선)
3. **이미지 최적화** (WebP 포맷, 지연 로딩)

### 중기 (3-6개월)
1. **빌드 시스템 현대화** (Webpack 또는 Vite 도입)
2. **컴포넌트 시스템** 구축 (재사용 가능한 UI 컴포넌트)
3. **성능 최적화** (번들 분할, 캐싱 전략)

### 장기 (6개월+)
1. **Next.js 또는 Gatsby 마이그레이션** 고려
2. **헤드리스 CMS** 도입 검토
3. **PWA 기능** 추가 (오프라인 지원, 푸시 알림)