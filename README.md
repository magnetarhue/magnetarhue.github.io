# moojeokpixel.github.io

<head>
    <style>
        html, body {
            margin: 0;
            padding: 0;
            width: 100%;
            height: 100%;
            display: block;
        }
        #canvas {
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            text-align: center;
            position: fixed;
            position: relative;
        }
        
        .error {
            font-family: Consolas;
            font-size: 1.2em;
            color: black;
            box-sizing: border-box;
            background-color: lightcoral;
            border-radius: 2px;
            border-color: lightblue;
            border-width: thin;
            border-style: solid;
            line-height: 1.4em;
            cursor:pointer;
        }
        .error:hover {
            color: black;
            background-color: brown;
            border-color: blue;
        }
        #message {
            font-family: Consolas;
            font-size: 1.2em;
            color: #ccc;
            background-color: black;
            font-weight: bold;
            z-index: 2;
            position: absolute;
        }

        #dat_gui_container {
            position: absolute;
            left: 0px;   /* position inside relatively positioned parent */
            top: 0px;
            z-index: 3;   /* adjust as needed */
        }

        /* Pause Button Style */
        
        /* Screenshot Button Style */
    </style>
</head>
<body>
    <div id="message"></div>
    <div id="dat_gui_container"></div>
    <div id="container">
        <!-- Pause Element -->
    </div>
    <!-- Screenshot Element -->
</body>
<script src="https://ajax.googleapis.com/ajax/libs/jquery/3.4.1/jquery.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/110/three.min.js"></script>
<!-- Stats.js -->
<script src='https://cdnjs.cloudflare.com/ajax/libs/stats.js/r16/Stats.min.js' onload='
let stats = new Stats();
compileTimePanel = stats.addPanel(new Stats.Panel('CT MS', '#ff8', '#221'));
stats.showPanel(1);
document.body.appendChild(stats.dom);
requestAnimationFrame(function loop() {
    stats.update();
    requestAnimationFrame(loop);
});
'></script>
<!-- dat.gui -->

<canvas id="canvas"></canvas>

<!-- Shaders -->

<script id='realisticeye_expeiment_magnetar.txt' type='x-shader/x-fragment'>
uniform vec3        iResolution;
uniform float       iTime;
uniform float       iTimeDelta;
uniform int         iFrame;
uniform vec4        iDate;
uniform vec3        iChannelResolution[10];
uniform vec4        iMouse;
uniform vec4        iMouseButton;
uniform sampler2D   iChannel0;
uniform sampler2D   iChannel1;
uniform sampler2D   iChannel2;
uniform sampler2D   iChannel3;
uniform sampler2D   iChannel4;
uniform sampler2D   iChannel5;
uniform sampler2D   iChannel6;
uniform sampler2D   iChannel7;
uniform sampler2D   iChannel8;
uniform sampler2D   iChannel9;
uniform sampler2D   iKeyboard;
uniform float       iSampleRate;

#define iGlobalTime iTime
#define iGlobalFrame iFrame

#define SHADER_TOY


// Created by inigo quilez - iq/2014
// License Creative Commons Attribution-NonCommercial-ShareAlike 3.0 Unported License.

	
float hash(vec2 p)
{
    p  = 50.0*fract( p*0.3183099);
    return fract( p.x*p.y*(p.x+p.y) );
}

float noise( in vec2 p )
{
    vec2 i = floor( p );
    vec2 f = fract( p );
	
	vec2 u = f*f*(3.0-2.0*f); // 물결
    //vec2 u = f*f*(3.0-2.0*f);

    return -1.0+2.0*mix( mix( hash( i + vec2(0.0,0.0) ), 
                              hash( i + vec2(1.0,0.0) ), u.x),
                         mix( hash( i + vec2(0.0,1.0) ), 
                              hash( i + vec2(1.0,1.0) ), u.x), u.y);
}

vec2 sd2Segment( vec3 a, vec3 b, vec3 p )
{
	vec3  pa = p - a;
	vec3  ba = b - a;
	float t = clamp( dot(pa,ba)/dot(ba,ba), 0.0, 1.0 );
	vec3  v = pa - ba*t;
	return vec2( dot(v,v), t );
}

float udBox(vec3 p, vec3 b)
{
    return length(max(abs(p)-b, 0.0));
}

float udRoundBox( vec3 p, vec3 b, float r )
{
    return length(max(abs(p)-b,0.0))-r;
}

float smin( float a, float b, float k )
{
	float h = clamp( 0.5 + 0.5*(b-a)/k, 0.0, 1.0 );
	return mix( b, a, h ) - k*h*(1.0-h);
}

//-----------------------------------------------------------------------------------

#define NUMI 11
#define NUMF 11.0

vec3 fishPos;
float fishTime;
float isJump;
float isJump2;

vec2 anima( float ih, float t )
{
    float an1 = 0.9*(0.5+0.2*ih)*cos(5.0*ih - 3.0*t + 6.2831/4.0);
    float an2 = 1.0*cos(3.5*ih - 1.0*t + 6.2831/4.0);
    float an = mix( an1, an2, isJump );
    float ro = 0.2*cos(4.0*ih - 1.0*t)*(1.0-0.5*isJump);
	return vec2( an, ro );
}

vec3 anima2( void )
{
    vec3 a1 = vec3(0.0,        sin(3.0*fishTime+6.2831/4.0),0.0);
    vec3 a2 = vec3(0.0,1.5+2.5*cos(1.0*fishTime),0.0);
	vec3 a = mix( a1, a2, isJump );
	a.y *= 0.5;
	a.x += 0.1*sin(0.1 - 1.0*fishTime)*(1.0-isJump);
    return a;
}

float sdDolphinCheap( vec3 p )
{
    float res = 100000.0;

	p -= fishPos;
		
	vec3 a = anima2();
	
	for( int i=0; i<NUMI; i++ )
	{	
		float ih = float(i)/NUMF;
		vec2 anim = anima( ih, fishTime );
		
		float ll = 0.48; if( i==0 ) ll=0.655; // 속도 ?
		vec3 b = a + ll*normalize(vec3(sin(anim.y), sin(anim.x), cos(anim.x)));
		
		vec2 dis = sd2Segment( a, b, p );

        float h = ih+dis.y/NUMF;
		float ra = 0.04 + h*(1.0-h)*(1.0-h)*2.7;
		res = min( res, sqrt(dis.x) - ra );
		
		a = b;
	}

	return 0.75 * res;
}

vec3 ccd, ccp;
	
vec2 sdDolphin( vec3 p )
{
    vec2 res = vec2( 1000.0, 0.0 );

	p -= fishPos;

	vec3 a = anima2();
	
	float or = 0.0;
	float th = 0.0;
	float hm = 0.0;

	vec3 p1 = a; vec3 d1=vec3(0.0);
	vec3 p2 = a; vec3 d2=vec3(0.0);
	vec3 p3 = a; vec3 d3=vec3(0.0);
	vec3 mp = a;
	for( int i=0; i<NUMI; i++ )
	{	
		float ih = float(i)/NUMF;
		vec2 anim = anima( ih, fishTime );
		float ll = 0.42; if( i==0 ) ll=0.655; // 몸통 길이 
		vec3 b = a + ll*normalize(vec3(sin(anim.y), sin(anim.x), cos(anim.x)));
		
		vec2 dis = sd2Segment( a, b, p );

		if( dis.x<res.x ) {res=vec2(dis.x,ih+dis.y/NUMF); mp=a+(b-a)*dis.y; ccd = b-a;}
		
		if( i==3 ) { p1=a; d1 = b-a; }
		if( i==4 ) { p3=a; d3 = b-a; }
		if( i==(NUMI-1) ) { p2=b; d2 = b-a; }

		a = b;
	}
	ccp = mp;
	
	float h = res.y;
	float ra = 0.05 + h*(1.0-h)*(1.0-h)*2.7;
	ra += 7.0*max(0.0,h-0.04)*exp(-30.0*max(0.0,h-0.04)) * smoothstep(-0.1, 0.1, p.y-mp.y);
	ra -= 0.03*(smoothstep(0.0, 0.1, abs(p.y-mp.y)))*(1.0-smoothstep(0.0,0.1,h));
	ra += 0.05*clamp(1.0-3.0*h,0.0,1.0);
    ra += 0.035*(1.0-smoothstep( 0.0, 0.025, abs(h-0.1) ))* (1.0-smoothstep(0.0, 0.1, abs(p.y-mp.y)));
	
	// body
	res.x = 0.75 * (distance(p,mp) - ra);
	//res.x = 0.75 * (res.x*3.0 - ra);

    // fin	
	d3 = normalize(d3);
	float k = sqrt(1.0 - d3.y*d3.y);
	mat3 ms = mat3(  d3.z/k, -d3.x*d3.y/k, d3.x,
				        0.0,            k, d3.y,
				    -d3.x/k, -d3.y*d3.z/k, d3.z );
	vec3 ps = p - p3;
	ps = ms*ps;
	ps.z -= 0.1;
    float d5 = length(ps.yz) - 0.9;
	d5 = max( d5, -(length(ps.yz-vec2(0.6,0.0)) - 0.35) );
	d5 = max( d5, udRoundBox( ps+vec3(0.0,-0.5,0.5), vec3(0.0,0.5,0.5), 0.02 ) );
	res.x = smin( res.x, d5, 0.1 );

	
#if 1
    // fin	
	d1 = normalize(d1);
	k = sqrt(1.0 - d1.y*d1.y);
	ms = mat3(  d1.z/k, -d1.x*d1.y/k, d1.x,
				   0.0,            k, d1.y,
               -d1.x/k, -d1.y*d1.z/k, d1.z );
	ps = p - p1;
	ps = ms*ps;
	ps.x = abs(ps.x);
	float l = ps.x;
	l=clamp( (l-0.4)/0.5, 0.0, 1.0 );
	l=4.0*l*(1.0-l);
	l *= 1.0-clamp(5.0*abs(ps.z+0.2),0.0,1.0);
	ps.xyz += vec3(-0.2,0.36,-0.2);
    d5 = length(ps.xz) - 0.8;
	d5 = max( d5, -(length(ps.xz-vec2(0.2,0.4)) - 0.8) );
	d5 = max( d5, udRoundBox( ps+vec3(0.0,0.0,0.0), vec3(1.0,0.0,1.0), 0.015+0.05*l ) );
	res.x = smin( res.x, d5, 0.12 );
#endif
	
    // tail	
	d2 = normalize(d2);
	mat2 mf = mat2( d2.z, d2.y, -d2.y, d2.z );
	vec3 pf = p - p2 - d2*0.25;
	pf.yz = mf*pf.yz;
    float d4 = length(pf.xz) - 0.6;
	d4 = max( d4, -(length(pf.xz-vec2(0.0,0.8)) - 0.9) );
	d4 = max( d4, udRoundBox( pf, vec3(1.0,0.005,1.0), 0.005 ) );
	res.x = smin( res.x, d4, 0.1 );
	
	
	return res;
}

const mat2 m2 = mat2( 0.80, -0.60, 0.60, 0.80 );

float almostIdentity( float x, float m, float n )
{
    if( x>m ) return x;
    float a = 2.0*n - m;
    float b = 2.0*m - 3.0*n;
    float t = x/m;
    return (a*t + b)*t*t + n;
}

float almostAbs( float x )
{
    return almostIdentity(abs(x), 0.05, 0.025 );
}

vec2 sdWaterCheap( vec3 p )
{
    vec2 q = 0.00*p.xz; // 파도잔물결정도 
    //    vec2 q = 0.05*p.xz;
	float f = 1.0; // 수면 경계 
    f += 0.50000*almostAbs(noise( q )); q = m2*q*2.02; q -= 0.1*iTime;
    f += 0.25000*almostAbs(noise( q )); q = m2*q*2.03; q += 0.2*iTime;
    f += 0.12500*almostAbs(noise( q )); q = m2*q*2.01; q -= 0.4*iTime;
    f += 0.06250*almostAbs(noise( q )); q = m2*q*2.02; q += 1.0*iTime;
    f += 0.03125*almostAbs(noise( q ));
    return vec2(3.7-4.0*f,f);
    //return vec2(4.1-6.0*f,f);
}

vec3 sdWater( vec3 p )
{
	vec2 q = 2.05*p.xz;

    vec2 w = sdWaterCheap( p );
     // 배경
	float sss = abs(sdDolphinCheap(p));
	float spla = exp(-4.0*sss); //	float spla = exp(-4.0*sss);
	spla += 0.5*exp(-14.0*sss);
	spla *= mix(1.0,texture( iChannel0, 0.2*p.xz ).x,spla*spla);
	spla *= -0.85;
	spla *= isJump;
	spla *= mix( 1.0, smoothstep(0.0,0.5,p.z-fishPos.z-1.5), isJump2 );

	return vec3( p.y-w.x + spla, w.y, sss );
}


vec2 intersectDolphin( in vec3 ro, in vec3 rd )
{
	const float maxd = 10.0;
	const float precis = 0.001;
    float t = 0.0;
	float l = 0.0;
    for( int i=0; i<128; i++ )
    {
	    vec2 res = sdDolphin( ro+rd*t );
        float h = res.x;
		l = res.y;
		if( h<precis || t>maxd ) break;
        t += h;
    }

    if( t>maxd ) t=-1.0;
    return vec2( t, l);
}

vec3 intersectWater( vec3 ro, in vec3 rd )
{
	const float precis = 0.001;
	float l = 0.0;
	float s = 0.0;

	float t = (-2.5-ro.y)/rd.y;  // 물 배경 위치? 
    //	float t = (2.5-ro.y)/rd.y; 
	if( t<0.0 ) return vec3(-1.0);

	for( int i=0; i<12; i++ )
    {
	    vec3 res = sdWater( ro+rd*t );
		l = res.y;
		s = res.z;
		if( abs(res.x)<precis ) break;
        t += res.x;
    }

    return vec3( t, l, s );
}

// http://iquilezles.org/www/articles/normalsSDF/normalsSDF.htm
vec3 calcNormalFish( in vec3 pos )
{
#if 0    
    const vec3 eps = vec3(0.08,0.0,0.0);
	float v = sdDolphin(pos).x;
	return normalize( vec3(
           sdDolphin(pos+eps.xyy).x - v,
           sdDolphin(pos+eps.yxy).x - v,
           sdDolphin(pos+eps.yyx).x - v ) );
#else
    #define ZERO (min(iFrame,0))

    // inspired by tdhooper and klems - a way to prevent the compiler from inlining map() 4 times
    vec3 n = vec3(0.8); //0.0 // 몸통 글로시 
    for( int i=ZERO; i<4; i++ )
    {
        vec3 e = 0.5773*(2.0*vec3((((i+3)>>1)&1),((i>>1)&1),(i&1))-1.0);
        n += e*sdDolphin(pos+0.08*e).x;
    }
    return normalize(n);
#endif    
}

vec3 calcNormalWater( in vec3 pos )
{
    const vec3 eps = vec3(0.025,0.0,0.0);
    float v = sdWater(pos).x;	
	return normalize( vec3( sdWater(pos+eps.xyy).x - v,
                            eps.x,
                            sdWater(pos+eps.yyx).x - v ) );
}

// http://iquilezles.org/www/articles/rmshadows/rmshadows.htm
float softshadow( in vec3 ro, in vec3 rd, float mint, float k )
{
    float res = 1.0;
    float t = mint;
	float h = 1.0;
    for( int i=0; i<25; i++ )
    {
        h = sdDolphinCheap(ro + rd*t);
        res = min( res, k*h/t );
		t += clamp( h, 0.05, 0.5 );
		if( h<0.0001 ) break;
    }
    return clamp(res,0.0,1.0);
}

const vec3 lig = vec3(0); // 빛색깔

vec3 doLighting( in vec3 pos, in vec3 nor, in vec3 rd, float glossy, float glossy2, float shadows, in vec3 col, float occ )
{
    vec3 hal = normalize(lig-rd);
	vec3 ref = reflect(rd,nor);
	
    // lighting
    float sky = clamp(nor.y,0.0,1.0);
	float bou = clamp(-nor.y,0.0,1.0);
    float dif = max(dot(nor,lig),0.0);
    float bac = max(0.3 + 0.7*dot(nor,-vec3(lig.x,0.0,lig.z)),0.0);
    float sha = 1.0-shadows; if( (shadows*dif)>0.001 ) sha=softshadow( pos+0.01*nor, lig, 0.0005, 32.0 );
    float fre = pow( clamp( 1.0 + dot(nor,rd), 0.0, 1.0 ), 5.0 );
    float spe = max( 0.0, pow( clamp( dot(hal,nor), 0.0, 1.0), 0.01+glossy ) );
    float sss = pow( clamp( 1.0 + dot(nor,rd), 0.0, 1.0 ), 2.0 );

    
    float shr = 1.0;
    //if( shadows>0.0 ) shr=softshadow( pos+0.01*nor, normalize(ref+vec3(0.0,1.0,0.0)), 0.0005, 8.0 );

    
    // lights // 빛 위치를 마우스 위치로 옮기기
    vec3 brdf = vec3(0.0);
    brdf += 20.0*dif*vec3(2.00,3.90,4.60)*vec3(sha,sha*0.5+0.5*sha*sha,sha*sha); //
    //    brdf += 20.0*dif*vec3(2.00,4.20,3.40)*vec3(sha,sha*0.5+0.5*sha*sha,sha*sha);
    brdf += 11.0*sky*vec3(0.20,0.40,0.55)*(0.5+0.5*occ);
    brdf += 1.0*bac*vec3(0.40,0.60,0.70);//*occ;
    brdf += 11.0*bou*vec3(0.05,0.30,0.50);
    brdf += 5.0*sss*vec3(0.40,0.40,0.40)*(0.3+0.7*dif*sha)*glossy*occ;
    brdf += 0.8*spe*vec3(1.30,1.00,0.90)*sha*dif*(0.1+0.9*fre)*glossy*glossy;
    brdf += shr*60.0*glossy*vec3(0.9,0.95,1.0)*occ*smoothstep( -0.2+0.2*glossy2, 0.2, ref.y )*(0.5+0.5*smoothstep( -0.2+0.2*glossy2, 1.0, ref.y ))*(0.04+0.96*fre);
    col = col*brdf;
    col += shr*(0.3 + 1.7*fre)*occ*glossy2*glossy2*40.0*vec3(1.0,0.9,0.8)*smoothstep( 0.0, 0.2, ref.y )*(0.5+0.5*smoothstep( 0.0, 1.0, ref.y ));//*smoothstep(-0.1,0.0,dif);
    col += 1.2*glossy*pow(spe,4.0)*vec3(1.4,1.1,0.9)*sha*dif*(0.04+0.96*fre)*occ;
	
	return col;
}

vec3 normalMap( in vec2 pos )
{
	pos *= 2.0;
	
	float v = texture( iChannel3, 0.015*pos ).x;
	vec3 nor = vec3( texture( iChannel3, 0.015*pos+vec2(1.0/1024.0,0.0)).x - v,
	                 1.0/16.0,
	                 texture( iChannel3, 0.015*pos+vec2(0.0,1.0/1024.0)).x - v );
	nor.xz *= -1.0;
	return normalize( nor );
}

void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
	vec2 q = fragCoord.xy / iResolution.xy;
    vec2 p = -1.0 + 2.0 * q;
    p.x *= iResolution.x/iResolution.y;
    vec2 m = vec2(0.5);
	if( iMouse.z>0.0 ) m = iMouse.xy/iResolution.xy;


    //-----------------------------------------------------
    // animate
    //-----------------------------------------------------
	
	fishTime = 0.6 + 2.0*iTime - 20.0;
	
	fishPos = vec3( 0.0, 0.0-0.2, -1.1*fishTime );
	
	isJump  = 0.5 + 0.5*cos(     -0.4+0.5*fishTime);
	isJump2 = 0.5 + 0.5*cos( 0.6+0.5*fishTime);
	float isJump3 = 0.5 + 0.5*cos(-1.4+0.5*fishTime);

    //-----------------------------------------------------
    // camera
    //-----------------------------------------------------

	float an = 1.2 + 0.1*iTime - 12.0*(m.x-0.5);

	vec3 ta = vec3(fishPos.x,0.8,fishPos.z) - vec3(0.0,0.0,-2.0);
	vec3 ro = ta + vec3(4.0*sin(an),3.1,4.0*cos(an));

    // shake
	ro += 0.5*sin(4.0*iTime*vec3(1.1,1.2,1.3)+vec3(3.0,0.0,1.0) );
	ta += 0.05*sin(4.0*iTime*vec3(1.7,1.5,1.6)+vec3(1.0,2.0,1.0) );

    // camera matrix
    vec3 ww = normalize( ta - ro );
    //vec3 uu = normalize( cross(ww,vec3(0.0,.0,0.0) ) );
	vec3 uu = normalize( vec3(-ww.z,0.0,ww.x) );
    vec3 vv = normalize( cross(uu,ww));
	
	// create view ray 카메라 뷰 확대/축소
	vec3 rd = normalize( p.x*uu + p.y*vv + 0.8*ww*(1.0+0.7*smoothstep(-1.4,7.4,sin(0.34*iTime))) ); //0.8
//	vec3 rd = normalize( p.x*uu + p.y*vv + 2.0*ww*(1.0+0.7*smoothstep(-0.4,0.4,sin(0.34*iTime))) );

    //-----------------------------------------------------
	// render
    //-----------------------------------------------------

	float t = 1000.0;
	
	vec3 col = vec3(0); // 배경 색 //vec3 col = vec3(0, 255, 0); 
	vec3 bgcol = vec3(0.6,0.7,0.8) - .2*clamp(rd.y,0.0,1.0);

    // quick step till y=3.2 bounding plane
	float pt = (3.2-ro.y)/rd.y;
	if( rd.y<0.0 && pt>0.0 ) ro=ro+rd*pt;

	// raymarch
    vec2 tmat1 = intersectDolphin(ro,rd);
	vec3 posy = vec3(-100000.0);
    if( tmat1.x>0.0 )
    {
		vec2 tmat = tmat1;
		t = tmat.x;
        // geometry
        vec3 pos = ro + tmat.x*rd;
        vec3 nor = calcNormalFish(pos);
		vec3 ref = reflect( rd, nor );
		vec3 fpos = pos - fishPos;

		vec3 auu = normalize( vec3(-ccd.z,0.0,ccd.x) );
		vec3 avv = normalize( cross(ccd,auu) );
		vec3 ppp = vec3( dot(fpos-ccp,auu),  dot(fpos-ccp,avv),  tmat.y );
		vec2 uv = vec2( 1.0*atan(ppp.x,ppp.y)/3.1416, 4.0*ppp.z );

		vec3 bnor = -1.0+2.0*texture(iChannel0,uv).xyz;
        nor += 0.01*bnor;

		vec3 te = texture( iChannel2, uv ).xyz;
		vec4 mate;
		mate.w = 10.0;
        // 색상 
        mate.xyz = mix( vec3(3.3,4.68, 10.26)*0.6, vec3(8.6,8.8,8.8), smoothstep(-0.05,0.05,ppp.y-tmat.y*0.5+0.1) );
        // 분홍   mate.xyz = mix( vec3(0.7,0.48,0.66)*0.6, vec3(0.6,0.9,1.0), smoothstep(-0.05,0.05,ppp.y-tmat.y*0.5+0.1) );
        // 디폴트   mate.xyz = mix( vec3(0.3,0.38,0.46)*0.6, vec3(0.8,0.9,1.0), smoothstep(-0.05,0.05,ppp.y-tmat.y*0.5+0.1) );
        mate.xyz *= 1.0 + 0.3*te;
		mate.xyz *= smoothstep( 0.0, 0.06, distance(vec3(abs(ppp.x),ppp.yz)*vec3(1.0,1.0,4.0),vec3(0.35,0.0,0.4)) );
		mate.xyz *= 1.0 - 0.75*(1.0-smoothstep( 0.0, 0.02, abs(ppp.y) ))*(1.0-smoothstep( 0.07, 0.11, tmat.y ));
		
		mate.xyz *= 0.08*0.13*0.6;
		
        mate.w *= (0.7+0.3*te.x)*smoothstep( 0.0, 0.01, pos.y-sdWaterCheap( pos ).x );
			
        // surface-light interacion
        col = doLighting( pos, nor, rd, mate.w, 0.0, 0.0, mate.xyz, 1.0 );
	
		posy = pos;
	}

	
    vec3 tmat2 = intersectWater(ro,rd);
	vec3 col2 = vec3(0.0);
	if( tmat2.x>0.0 && (tmat1.x<0.0 || tmat2.x<tmat1.x) )
	{
		vec3 tmat = tmat2;

        t = tmat.x;

        vec3 pos = ro + tmat.x*rd;
        vec3 nor = calcNormalWater(pos);
		vec3 ref = reflect( rd, nor );
        nor = normalize( nor + 0.15*normalMap(pos.xz) );
        float fre = pow( clamp(1.0 + dot( rd, nor ),0.0,1.0), 2.0 );
        
        
        vec4 mate = vec4(0.4,0.4,0.4,0.0);
		mate.xyz = 0.05*mix( vec3(0.0,0.07,0.2)*0.8, vec3(0.0,0.12,0.2), (1.0-smoothstep(0.2,0.8,tmat.y))*(0.5+0.5*fre) );
		mate.w = fre;	

        float foam = 1.0-smoothstep( 0.4, 0.6, tmat.y );
        foam *= abs(nor.z)*2.0;
        foam *= clamp(1.0-2.0*texture( iChannel2, vec2(1.0,0.75)*0.1*pos.xz ).x*2.0,0.0,1.0);
        mate = mix( mate, vec4(0.1*0.2,0.11*0.2,0.13*0.2,0.5), foam );
        
		float al = clamp( 0.5 + 0.2*(pos.y - posy.y), 0.0, 1.0 );
		
		foam = exp( -3.0*abs(tmat.z) );
		foam *= texture( iChannel3, pos.zx ).x;
		foam = clamp( foam*3.0, 0.0, 1.0 );
		foam *= isJump;
		foam *= mix( 1.0, smoothstep(0.0,0.0,pos.z-fishPos.z-1.5), isJump2 );
		mate.xyz = mix( mate.xyz, vec3(0.0,0.0,0.0)*0.05, foam*foam );
        
		col = mix( col, vec3(0.9,0.95,1.0)*1.2, foam );
		al *= 0.0-foam; // 배경 없애기 = 0

	//	al *= vec3(0, 255, 0); 

		float occ = clamp(3.5*sdDolphinCheap(pos+vec3(0.0,0.4,0.0)) * sdDolphinCheap(pos+vec3(0.0,1.0,0.0)),0.0,1.0);
        occ = mix(1.0,occ,isJump);
        occ = 0.35 + 0.65*occ;
		mate.xyz *= occ;
        col *= occ;

		mate.xyz = doLighting( pos, nor, rd, mate.w*10.0, mate.w*0.5, 1.0, mate.xyz, occ );
		
        // caustics
        float cc  = 0.65*texture( iChannel0, 2.5*0.02*posy.xz + 0.007*iTime*vec2( 1.0, 0.0) ).x;
        cc += 0.35*texture( iChannel0, 1.8*0.04*posy.xz + 0.011*iTime*vec2( 0.0, 1.0) ).x;
        cc = 0.6*(1.0-smoothstep( 0.0, 0.05, abs(cc-0.5))) + 
	         0.4*(1.0-smoothstep( 0.0, 0.20, abs(cc-0.5)));
        col *= 1.0 + 0.8*cc;
		
		col = mix( col, mate.xyz, al );
        
	}
		
	
	float sun = pow( max(0.0,dot( lig, rd )),8.0 );
	col += vec3(1.8,0.5,1.1)*sun*0.3;

    // gamma
	col = pow( clamp(col,0.0,1.0), vec3(0.45) );

    // color
    //col = col*0.7 + 0.3*col*col*(3.0-2.0*col);
    col = col*0.5 + 0.5*col*col*(3.0-2.0*col);
		
    // vigneting	
	col *= 0.5 + 0.5*pow( 16.0*q.x*q.y*(1.0-q.x)*(1.0-q.y), 0.1 );
    //	col *= 0.5 + 0.5*pow( 16.0*q.x*q.y*(1.0-q.x)*(1.0-q.y), 0.1 );

    // fade	
	col *= smoothstep( 0.0, 1.0, iTime );
	
	fragColor = vec4( col, 1.0 );
}
void main() {
    mainImage(gl_FragColor, gl_FragCoord.xy);
}
</script>

<script type="text/javascript">
    let vscode = undefined;
    if (typeof acquireVsCodeApi === 'function') {
        vscode = acquireVsCodeApi();
    }
    var compileTimePanel;

    let revealError = function(line, file) {
        if (vscode) {
            vscode.postMessage({
                command: 'showGlslsError',
                line: line,
                file: file
            });
        }
    };

    let currentShader = {};
    // Error Callback
    console.error = function () {
        if('7' in arguments) {
            let errorRegex = /ERROR: \d+:(\d+):\W(.*)\n/g;
            let rawErrors = arguments[7];
            let match;
            
            let diagnostics = [];
            let message = '';
            while(match = errorRegex.exec(rawErrors)) {
                let lineNumber = Number(match[1]) - currentShader.LineOffset;
                let error = match[2];
                diagnostics.push({
                    line: lineNumber,
                    message: error
                });
                let lineHighlight = `<a class='error' unselectable onclick='revealError(${lineNumber}, "${currentShader.File}")'>Line ${lineNumber}</a>`;
                message += `<li>${lineHighlight}: ${error}</li>`;
            }
            console.log(message);
            let diagnosticBatch = {
                filename: currentShader.File,
                diagnostics: diagnostics
            };
            if (vscode !== undefined) {
                vscode.postMessage({
                    command: 'showGlslDiagnostic',
                    type: 'error',
                    diagnosticBatch: diagnosticBatch
                });
            }
    
            $('#message').append(`<h3>Shader failed to compile - ${currentShader.Name} </h3>`);
            $('#message').append('<ul>');
            $('#message').append(message);
            $('#message').append('</ul>');
        }
    };

    // Development feature: Output warnings from third-party libraries
    // console.warn = function (message) {
    //     $("#message").append(message + '<br>');
    // };

    let clock = new THREE.Clock();
    let pausedTime = 0.0;
    let deltaTime = 0.0;
    let startingTime = 0;
    let time = startingTime;

    let date = new THREE.Vector4();

    let updateDate = function() {
        let today = new Date();
        date.x = today.getFullYear();
        date.y = today.getMonth();
        date.z = today.getDate();
        date.w = today.getHours() * 60 * 60 
            + today.getMinutes() * 60
            + today.getSeconds()
            + today.getMilliseconds() * 0.001;
    };
    updateDate();

    let paused = false;
    let pauseButton = document.getElementById('pause-button');
    if (pauseButton) {
        pauseButton.onclick = function(){
            paused = pauseButton.checked;
            if (!paused) {
                // Audio Resume
                pausedTime += clock.getDelta();
            }
            else {
                // Audio Pause
            }
        };
    }
    
    {
        let screenshotButton = document.getElementById("screenshot");
        if (screenshotButton) {
            screenshotButton.addEventListener('click', saveScreenshot);
        }
    }

    let canvas = document.getElementById('canvas');
    let gl = canvas.getContext('webgl2');
    let isWebGL2 = gl != null;
    if (gl == null) gl = canvas.getContext('webgl');
    let supportsFloatFramebuffer = (gl.getExtension('EXT_color_buffer_float') != null) || (gl.getExtension('WEBGL_color_buffer_float') != null);
    let supportsHalfFloatFramebuffer = (gl.getExtension('EXT_color_buffer_half_float') != null);
    let framebufferType = THREE.UnsignedByteType;
    if (supportsFloatFramebuffer) framebufferType = THREE.FloatType;
    else if (supportsHalfFloatFramebuffer) framebufferType = THREE.HalfFloatType;

    let renderer = new THREE.WebGLRenderer({ canvas: canvas, antialias: true, context: gl, preserveDrawingBuffer: true });
    let resolution = new THREE.Vector3();
    let mouse = new THREE.Vector4(1112, 661, -330, -660);
    let mouseButton = new THREE.Vector4(0, 0, 0, 0);
    let normalizedMouse = new THREE.Vector2(0.24413145539906103, 0.9988610478359908);
    let frameCounter = 0;

    // Audio Init
    const audioContext = {
        sampleRate: 0
    };
    // Audio Resume

    let buffers = [];
    // Buffers
    buffers.push({
        Name: 'realisticeye_expeiment_magnetar.txt',
        File: 'c:/Users/user/Desktop/vscode/realisticeye_expeiment_magnetar.txt',
        LineOffset: 133,
        Target: null,
        ChannelResolution: Array(10).fill(new THREE.Vector3(0,0,0)),
        PingPongTarget: null,
        PingPongChannel: 0,
        Dependents: [],
        Shader: new THREE.ShaderMaterial({
            fragmentShader: document.getElementById('realisticeye_expeiment_magnetar.txt').textContent,
            depthWrite: false,
            depthTest: false,
            uniforms: {
                iResolution: { type: 'v3', value: resolution },
                iTime: { type: 'f', value: 0.0 },
                iTimeDelta: { type: 'f', value: 0.0 },
                iFrame: { type: 'i', value: 0 },
                iMouse: { type: 'v4', value: mouse },
                iMouseButton: { type: 'v2', value: mouseButton },
    
                iChannelResolution: { type: 'v3v', value: Array(10).fill(new THREE.Vector3(0,0,0)) },
    
                iDate: { type: 'v4', value: date },
                iSampleRate: { type: 'f', value: audioContext.sampleRate },
    
                iChannel0: { type: 't' },
                iChannel1: { type: 't' },
                iChannel2: { type: 't' },
                iChannel3: { type: 't' },
                iChannel4: { type: 't' },
                iChannel5: { type: 't' },
                iChannel6: { type: 't' },
                iChannel7: { type: 't' },
                iChannel8: { type: 't' },
                iChannel9: { type: 't' },
    
                resolution: { type: 'v2', value: resolution },
                time: { type: 'f', value: 0.0 },
                mouse: { type: 'v2', value: normalizedMouse },
            }
        })
    });
    let commonIncludes = [];
    // Includes
    

    // WebGL2 inserts more lines into the shader
    if (isWebGL2) {
        for (let buffer of buffers) {
            buffer.LineOffset += 16;
        }
    }

    // Keyboard Init
    
    // Uniforms Init
    // Uniforms Update

    let texLoader = new THREE.TextureLoader();
    // Texture Init
    

    let scene = new THREE.Scene();
    let quad = new THREE.Mesh(
        new THREE.PlaneGeometry(resolution.x, resolution.y),
        null
    );
    scene.add(quad);
    
    let camera = new THREE.OrthographicCamera(-resolution.x / 2.0, resolution.x / 2.0, resolution.y / 2.0, -resolution.y / 2.0, 1, 1000);
    camera.position.set(0, 0, 10);

    // Run every shader once to check for compile errors
    let compileTimeStart = performance.now();
    let failed=0;
    for (let include of commonIncludes) {
        currentShader = {
            Name: include.Name,
            File: include.File,
            // add two for version and precision lines
            LineOffset: 26 + 2
        };
        // bail if there is an error found in the include script
        if(compileFragShader(gl, document.getElementById(include.Name).textContent) == false) {
            throw Error(`Failed to compile ${include.Name}`);
        }
    }

    for (let buffer of buffers) {
        currentShader = {
            Name: buffer.Name,
            File: buffer.File,
            LineOffset: buffer.LineOffset
        };
        quad.material = buffer.Shader;
        renderer.setRenderTarget(buffer.Target);
        renderer.render(scene, camera);
    }
    currentShader = {};
    let compileTimeEnd = performance.now();
    let compileTime = compileTimeEnd - compileTimeStart;
    if (compileTimePanel !== undefined) {
        for (let i = 0; i < 200; i++) {
            compileTimePanel.update(compileTime, 200);
        }
    }

    computeSize();
    render();

    function addLineNumbers( string ) {
        let lines = string.split( '\\n' );
        for ( let i = 0; i < lines.length; i ++ ) {
            lines[ i ] = ( i + 1 ) + ': ' + lines[ i ];
        }
        return lines.join( '\\n' );
    }

    function compileFragShader(gl, fsSource) {
        const fs = gl.createShader(gl.FRAGMENT_SHADER);
        gl.shaderSource(fs, fsSource);
        gl.compileShader(fs);
        if (!gl.getShaderParameter(fs, gl.COMPILE_STATUS)) {
            const fragmentLog = gl.getShaderInfoLog(fs);
            console.error( 'THREE.WebGLProgram: shader error: ', gl.getError(), 'gl.COMPILE_STATUS', null, null, null, null, fragmentLog );
            return false;
        }
        return true;
    }

    function render() {
        requestAnimationFrame(render);
        // Pause Whole Render
        if (paused) return;

        // Advance Time
        deltaTime = clock.getDelta();
        time = startingTime + clock.getElapsedTime() - pausedTime;
        updateDate();

        // Audio Update

        for (let buffer of buffers) {
            buffer.Shader.uniforms['iResolution'].value = resolution;
            buffer.Shader.uniforms['iTimeDelta'].value = deltaTime;
            buffer.Shader.uniforms['iTime'].value = time;
            buffer.Shader.uniforms['iFrame'].value = frameCounter;
            buffer.Shader.uniforms['iMouse'].value = mouse;
            buffer.Shader.uniforms['iMouseButton'].value = mouseButton;

            buffer.Shader.uniforms['resolution'].value = resolution;
            buffer.Shader.uniforms['time'].value = time;
            buffer.Shader.uniforms['mouse'].value = normalizedMouse;

            quad.material = buffer.Shader;
            renderer.setRenderTarget(buffer.Target);
            renderer.render(scene, camera);
        }
        
        // Uniforms Update

        // Keyboard Update

        for (let buffer of buffers) {
            if (buffer.PingPongTarget) {
                [buffer.PingPongTarget, buffer.Target] = [buffer.Target, buffer.PingPongTarget];
                buffer.Shader.uniforms[`iChannel${buffer.PingPongChannel}`].value = buffer.PingPongTarget.texture;
                for (let dependent of buffer.Dependents) {
                    const dependentBuffer = buffers[dependent.Index];
                    dependentBuffer.Shader.uniforms[`iChannel${dependent.Channel}`].value = buffer.Target.texture;
                }
            }
        }

        frameCounter++;
    }
    function computeSize() {
        let forceAspectRatio = (width, height) => {
            // Forced aspect ratio
            let forcedAspects = [0,0];
            let forcedAspectRatio = forcedAspects[0] / forcedAspects[1];
            let aspectRatio = width / height;

            if (forcedAspectRatio <= 0 || !isFinite(forcedAspectRatio)) {
                let resolution = new THREE.Vector3(width, height, 1.0);
                return resolution;
            }
            else if (aspectRatio < forcedAspectRatio) {
                let resolution = new THREE.Vector3(width, Math.floor(width / forcedAspectRatio), 1);
                return resolution;
            }
            else {
                let resolution = new THREE.Vector3(Math.floor(height * forcedAspectRatio), height, 1);
                return resolution;
            }
        };
        
        // Compute forced aspect ratio and align canvas
        resolution = forceAspectRatio(window.innerWidth, window.innerHeight);
        canvas.style.left = `${(window.innerWidth - resolution.x) / 2}px`;
        canvas.style.top = `${(window.innerHeight - resolution.y) / 2}px`;

        for (let buffer of buffers) {
            if (buffer.Target) {
                buffer.Target.setSize(resolution.x, resolution.y);
            }
            if (buffer.PingPongTarget) {
                buffer.PingPongTarget.setSize(resolution.x, resolution.y);
            }
        }
        renderer.setSize(resolution.x, resolution.y, false);
        
        // Update Camera and Mesh
        quad.geometry = new THREE.PlaneGeometry(resolution.x, resolution.y);
        camera.left = -resolution.x / 2.0;
        camera.right = resolution.x / 2.0;
        camera.top = resolution.y / 2.0;
        camera.bottom = -resolution.y / 2.0;
        camera.updateProjectionMatrix();

        // Reset iFrame on resize for shaders that rely on first-frame setups
        frameCounter = 0;
    }
    function saveScreenshot() {
        let doSaveScreenshot = () => {
            renderer.domElement.toBlob(function(blob){
                let a = document.createElement('a');
                let url = URL.createObjectURL(blob);
                a.href = url;
                a.download = 'shadertoy.png';
                a.click();
            }, 'image/png', 1.0);
        };

        let forcedScreenshotResolution = [0,0];
        if (forcedScreenshotResolution[0] <= 0 || forcedScreenshotResolution[1] <= 0) {
            renderer.render(scene, camera);
            doSaveScreenshot();
        }
        else {
            renderer.setSize(forcedScreenshotResolution[0], forcedScreenshotResolution[1], false);
            
            for (let buffer of buffers) {
                buffer.Shader.uniforms['iResolution'].value = new THREE.Vector3(forcedScreenshotResolution[0], forcedScreenshotResolution[1], 1);
                buffer.Shader.uniforms['resolution'].value = new THREE.Vector3(forcedScreenshotResolution[0], forcedScreenshotResolution[1], 1);

                quad.material = buffer.Shader;
                renderer.setRenderTarget(buffer.Target);
                renderer.render(scene, camera);
            }

            doSaveScreenshot();
            renderer.setSize(resolution.x, resolution.y, false);
        }
    }
    function updateMouse() {
        if (vscode !== undefined) {
            vscode.postMessage({
                command: 'updateMouse',
                mouse: {
                    x: mouse.x,
                    y: mouse.y,
                    z: mouse.z,
                    w: mouse.w
                },
                normalizedMouse: {
                    x: normalizedMouse.x,
                    y: normalizedMouse.y
                }
            });
        }
    }
    let dragging = false;
    function updateNormalizedMouseCoordinates(clientX, clientY) {
        let rect = canvas.getBoundingClientRect();
        let mouseX = clientX - rect.left;
        let mouseY = resolution.y - clientY - rect.top;

        if (mouseButton.x + mouseButton.y != 0) {
            mouse.x = mouseX;
            mouse.y = mouseY;
        }

        normalizedMouse.x = mouseX / resolution.x;
        normalizedMouse.y = mouseY / resolution.y;
    }
    canvas.addEventListener('mousemove', function(evt) {
        updateNormalizedMouseCoordinates(evt.clientX, evt.clientY);
        updateMouse();
    }, false);
    canvas.addEventListener('mousedown', function(evt) {
        if (evt.button == 0)
            mouseButton.x = 1;
        if (evt.button == 2)
            mouseButton.y = 1;

        if (!dragging) {
            updateNormalizedMouseCoordinates(evt.clientX, evt.clientY);
            mouse.z = mouse.x;
            mouse.w = mouse.y;
            dragging = true
        }

        updateMouse();
    }, false);
    canvas.addEventListener('mouseup', function(evt) {
        if (evt.button == 0)
            mouseButton.x = 0;
        if (evt.button == 2)
            mouseButton.y = 0;

        dragging = false;
        mouse.z = -mouse.z;
        mouse.w = -mouse.w;

        updateMouse();
    }, false);
    window.addEventListener('resize', function() {
        computeSize();
    });

    // Keyboard Callbacks
</script>,
