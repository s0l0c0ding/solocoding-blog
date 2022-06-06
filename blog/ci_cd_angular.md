---
published: true
title: Continuous integration and deployment in Angular/Scully applications
description: We will see how to implement the CI/CD concepts using Github actions and Netlify
date: 2020-08-17
author: Bassem
slug: eng_angular_angular_ci_cd_githupactions
photo: assets/stock/cicd.webp
imgCredit:
keywords:
  - angular
  - scully
  - devops
language: en
tweetId: '1295437685411151875'
output:
  html_document:
    css: post-details.component.css
---
In this post I am going to share with you how I implemented the CI/CD concepts to my personal blog built with JAMstack technique using Scully (static site generator for Angular) , Github and Netlify.
<br>
<br>
When we talk about CI, we mean continuous integration; it consists of running a test suite against every push or pull request to validate the written code. The primary goal is to keep our application bug free.
<br>
On the other hand CD stands for continuous deployment, the process to automate the deploy after a successful automated integration test phase. The benefit is bringing features/code changes quickly to production and earlier feedback from users.
<br>
<br>
**Continuous integration**
<br>
This phase can be divided into two steps, the first one is producing the test cases, eventually for each line of code. When using a test driven development methodology, you have to write the test before any line of code (very useful to reduce bugs and having a clear vision of the feature to  be implemented before coding).
In Angular, every time you create a component or a service using the CLI; a test will be automatically generated for you under the name component.spec.ts . The CLI will take care of Jasmine and Karma configuration for you.
<br>
Let’s see the test for the following component:
```typescript
export class DashboardComponent implements OnDestroy {

  keyword: string;
  subFilter: Subscription;
  linksFiltred$: Observable<any>;

  constructor(private scully: ScullyRoutesService, private route: ActivatedRoute) {
    this.subFilter = this.route.params.subscribe(params => {
      this.keyword = params['categoryId'];
      this.linksFiltred$ = this.scully.available$;
    });
  }

  ngOnDestroy(): void {
    this.subFilter.unsubscribe();
  }
 
  currentTag(link: any): boolean {
…
}
```
As you can see it’s a simple component, but with some dependencies which have to be mocked in the unit test. In the constructor I injected the ActivatedRoute to get the params associated with the current route and the ScullyRoutesService to have access to the available routes generated by Scully, specifically the ones related to the markdown files.
Here is the code to test the DashboardComponent:
```typescript
describe('DashboardComponent', () => {
  let component: DashboardComponent;
  let fixture: ComponentFixture<DashboardComponent>;
 
  beforeEach(() => {
    const link = {
      keywords:'angular',
       date: '2020-04-26'
    }
    const params = {
      categoryId: 'angular'
    }

    TestBed.configureTestingModule({
      declarations: [DashboardComponent],
      providers: [
        { provide: ScullyRoutesService, useValue: {
          available$: of([link])
        } },
        { provide: ActivatedRoute, useValue: {
          params: of(params)
        }}
      ]
    });
    
    fixture = TestBed.createComponent(DashboardComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should create', () => {
    expect(component).toBeTruthy;
  });

  it('should inject props', () => {
    expect(component.keyword).toEqual('angular');
  });

  it('should have <li> with "Angular"', () => {
    const bannerElement: HTMLElement = fixture.nativeElement;
    const p = bannerElement.querySelector('li');
    expect(p.textContent).toContain('Angular');
  });
});
```
Most of the above code is generated by the angular cli, let’s break it down:
<br>
*BeforeEach / TestBed:*
<br>
The `beforeEach()` is used to avoid code duplication for the TestBed configuration. This is the most important part, because we configure an instance of the component which will be tested.
<br>
As you saw in the component code, there are two services injected, which currently need to be mocked or stubbed to isolate our component. As you can see in the following snippet, I stubbed the two services (The main difference between mocks and stubs is that the mock can be configured during the execution of the test, on the other hand stubs are already configured with predetermined values):
```typescript
 TestBed.configureTestingModule({
      declarations: [DashboardComponent],
      providers: [
        { provide: ScullyRoutesService, useValue: {
          available$: of([link])
        } },
        { provide: ActivatedRoute, useValue: {
          params: of(params)
        }}
      ]
    });
```
For the ScullyRoutesService I configured the `available$` property with an observable of the object “link” using the “of” rxjs operator, did the same with the property params of the ActivatedRoute.
Once defined the testBed configuration we make an instance with the  [ComponentFixture](https://angular.io/guide/testing-components-basics#componentfixture).
<br>
Now in the following test , we are making sure that our component is instantiated:
```typescript
it('should create', () => {
    expect(component).toBeTruthy;
  });
```
A second test , is the following:
```typescript
it('should inject props', () => {
expect(component.keyword).toEqual('angular');
  });
```
Just making sure my stubs are injected. When you test the rendered HTML you have to make sure that you have called the detectChanges before the assertions (as we did in the beforeEach block):
```typescript
fixture.detectChanges()
```
To have a reliable test, you will have to set a coverage code percentage, the higher it is the better it's, a good average will be 80%. To enable the code coverage and the desired percentage, take a look at the official [Angular docs](https://angular.io/guide/testing-code-coverage).
<br>
Once the tests are ready and complete we need to run it against each pull request/push, hopefully locally first ( Always run the tests before pushing 😀 ) and then before approving the request.
<br>
<br>
**GitHup actions**
<br>
The last step in the continuous integration is running the developed test suite automatically, for this reason I built a custom githup action.
Here is my deployment.yml (must be placed in github/workflows at the root of your repository) , that I am using, it’s based on a standard node.js action:
```yml
name: Node.js CI

on:
  push:
    branches: [ dev, master ]
  pull_request:
    branches: [ master ]

jobs:
  build:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [12.x]

    steps:
    - uses: actions/checkout@v2

    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v1
      with:
        node-version: ${{ matrix.node-version }}

    - name: Cache node modules
      uses: actions/cache@v1
      env:
        cache-name: cache-node-modules
      with:
        path: ~/.npm
        key: ${{ runner.os }}-build-${{ env.cache-name }}-${{ hashFiles('**/package-lock.json') }}
        restore-keys: |
          ${{ runner.os }}-build-${{ env.cache-name }}-
          ${{ runner.os }}-build-
          ${{ runner.os }}-
    - name: Install Dependencies
      run: sudo npm install
    - name: Test
      run: npm run test -- --no-watch --no-progress --browsers=ChromeHeadlessCI
    - name: build app angular
      run: |
        npm run build -- --prod --stats-json
    - name: build static scully
      run: npm run scully -- --scanRoutes --showGuessError
```
 Let’s break it down , starting from the environment configuration:
 ```yml
on:
  push:
    branches: [ dev, master ]
  pull_request:
    branches: [ master ]
```
Here the action is triggered on every push on dev and master branches and with each pull request on the master branch (the trigger can be any supported [GitHub event](https://docs.github.com/en/actions/reference/events-that-trigger-workflows) ).
```yml
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [12.x]
```
In the above snippet I am deciding which OS and node version to be used in the workflow. Worth mentioning that GithupActions support also custom Docker containers.
And finally here are the steps of the workflow:
- actions/checkout@v2: using the built in action to clone the code;
- Use Node.js ${{ matrix.node-version }}: a standard action to install the desired node version referenced in the strategy configuration;
- Cache node modules: another built in action to cache the node modules to make the next workflows faster;
- Install Dependencies: just an npm install;
- Test: 
```yml
run: npm run test -- --no-watch --no-progress --browsers=ChromeHeadlessCI
```
We are using npm because @angular/cli is uninstalled, we don't need the web report, so we run it with no-watch, no progress flags and with a headless chrome browser. That last step need to be configured in the Karma configuration file, karma.conf.js:
```javascript
browsers: ['Chrome'],
    customLaunchers: {
      ChromeHeadlessCI: {
        base: 'ChromeHeadless',
        flags: ['--no-sandbox']
      }
```
- Build app angular: building our angular app, I am using the state-json flag, because it’s needed for a Scully plugin ([scully-plugin-disable-angular](https://github.com/samvloeberghs/kwerri-oss/tree/master/projects/scully-plugin-disable-angular));
- Build static scully: running the scully build with the scanRoutes flag to discover new posts/markdown files.
<br>
<br>

**Continuous deployment**
<br>
My blog is hosted on Netlify. It’s really easy to automate your deployments with this platform. Once your Github repo is connected to the platform with read and write access (needed to see checks, commit statuses and pull requests) and the deployment configuration is done correctly, the site will get built automatically with each commit. After authorization, Netlify will get listed under your repository’s integration tab on Github.
<br>
<br>
*Deployment settings*
<br>
There are few parameters to be setted: 
- Build command: 
  <br>
```npm run build -- --prod --stats-json && npm run scully -- --scanRoutes```
- Publish directory: dist/static/, as scully build the static assets into this directory ;
- Production branch: the branch to monitor for continuous builds, you can also set a deployment preview , if you want.
- Environment variables: NODE_VERSION:12.18.3, as Scully is working with this lts node version.

<br>
For sure there are other Netlify features, but this is enough for my deployment and usage case.
When I want to publish a new post, I just have to push it on the prod branch and the site gets built. When I include new features , first I get it done on the dev branch and upon successful test workflow I merge it into master and after staging into prod.
<br>
<br>
And that’s it, you are ready to go. I hope you find it useful. 


