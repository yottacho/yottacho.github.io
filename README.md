# yottacho.github.io

[Using Jekyll as a static site generator with GitHub Pages](https://help.github.com/articles/using-jekyll-as-a-static-site-generator-with-github-pages/)

## 로컬 서버에서 테스트하기
간단하게 테스트 할 용도

1. 루비 인스톨
```bash
yum install ruby
```

2. 루비 버전 관리를 위한 rvm 설치
```bash
curl -sSL https://rvm.io/mpapis.asc | gpg2 --import -
curl -sSL https://get.rvm.io | bash -s stable --ruby
source /usr/local/rvm/scripts/rvm
```

3. (필요시) 루비 버전 업그레이드
```bash
rvm install 2.3.0
rvm --default use 2.3.0
```

4. gem 으로 Jekyll 인스톨
```bash
gem install jekyll
gem install jekyll-coffeescript
gem install jekyll-sass-converter
gem install jekyll-mentions
gem install github-pages
```

5. (처음) GitHub clone
```bash
git clone "ssh://git@github.com/(id)/(id).github.io.git"
cd (id).github.io
```

6. (처음) Jekyll 초기화
```bash
jekyll new . --force
```

7. Jekyll 서버 구동
```bash
jekyll serve -w -H <IP addr>
# jekyll 서버는 127.0.0.1 아답터에 bind하는데, 외부IP를 가진 아답터에 bind해야 다른 PC에서 확인할 수 있다.
```
