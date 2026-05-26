import { useEffect, useRef, useState } from "react";
import * as THREE from "three";

// ═══════════════════════════════════════
// CONSTANTS
// ═══════════════════════════════════════
const BOARD = 8;
const TILE = 2.2;
const PIECES_DEF = {
  WIZARD:   { name:"קוסם",    emoji:"🧙", color:0xc084fc, hp:70,  atk:36, desc:"שורף +10 נזק" },
  KNIGHT:   { name:"אביר",    emoji:"⚔️",  color:0xfbbf24, hp:120, atk:28, desc:"מגן אחרי תקיפה" },
  ARCHER:   { name:"קשת",     emoji:"🏹", color:0x34d399, hp:80,  atk:30, desc:"מרעיל אויב" },
  DRAGON:   { name:"דרקון",   emoji:"🐉", color:0xf87171, hp:180, atk:44, desc:"מדהים אויב" },
  HEALER:   { name:"מרפא",    emoji:"💚", color:0x22d3ee, hp:60,  atk:12, desc:"מרפא עצמו +15" },
  ASSASSIN: { name:"מתנקש",   emoji:"🗡️", color:0xfb7185, hp:75,  atk:55, desc:"הגחה מאחורי" },
};

const DICE = [
  { label:"MISS",     icon:"💨", color:"#888",    desc:"מחמיץ!",        apply:a=>({dmg:0,        fx:"miss"    }) },
  { label:"NORMAL",   icon:"⚔️",  color:"#e2e8f0", desc:"נזק רגיל",      apply:a=>({dmg:a,        fx:"normal"  }) },
  { label:"DOUBLE",   icon:"✖️",  color:"#f59e0b", desc:"נזק x2!",       apply:a=>({dmg:a*2,      fx:"double"  }) },
  { label:"CRITICAL", icon:"💥", color:"#ef4444", desc:"קריטי x3!",     apply:a=>({dmg:a*3,      fx:"critical"}) },
  { label:"DRAIN",    icon:"🩸", color:"#a855f7", desc:"גונב חיים",     apply:a=>({dmg:a,        fx:"drain"   }) },
  { label:"BLOCK",    icon:"🛡️", color:"#60a5fa", desc:"חסום!",         apply:a=>({dmg:0,        fx:"block"   }) },
];

// ═══════════════════════════════════════
// GAME LOGIC
// ═══════════════════════════════════════
let _uid = 0;
const mkP = (type, color) => ({
  type, color, id: ++_uid,
  hp: PIECES_DEF[type].hp, maxHp: PIECES_DEF[type].hp,
  burning: false, poisoned: false, stunned: 0, shielded: false,
});

const initBoard = () => {
  const b = Array(BOARD).fill(null).map(() => Array(BOARD).fill(null));
  const back = ["ARCHER","KNIGHT","ASSASSIN","DRAGON","HEALER","WIZARD","KNIGHT","ARCHER"];
  const front = ["KNIGHT","ARCHER","KNIGHT","ARCHER","KNIGHT","ARCHER","KNIGHT","ARCHER"];
  back.forEach((t,i) => { b[0][i]=mkP(t,"black"); b[7][i]=mkP(t,"white"); });
  front.forEach((t,i) => { b[1][i]=mkP(t,"black"); b[6][i]=mkP(t,"white"); });
  return b;
};

const getMoves = (r, c, board, color) => {
  const m = [], p = board[r][c];
  if (!p) return m;
  const add = (nr, nc, atk=false) => {
    if (nr<0||nr>=BOARD||nc<0||nc>=BOARD) return false;
    if (board[nr][nc]) { if(board[nr][nc].color!==color&&atk) m.push({row:nr,col:nc,attack:true}); return false; }
    m.push({row:nr,col:nc}); return true;
  };
  const slide = (dirs) => dirs.forEach(([dr,dc])=>{ for(let i=1;i<BOARD;i++) if(!add(r+dr*i,c+dc*i,true))break; });

  if (p.type==="WIZARD")  slide([[-1,-1],[-1,1],[1,-1],[1,1]]);
  if (p.type==="ARCHER")  slide([[-1,0],[1,0],[0,-1],[0,1]]);
  if (p.type==="DRAGON"||p.type==="HEALER") {
    [[-1,0],[1,0],[0,-1],[0,1],[-1,-1],[-1,1],[1,-1],[1,1]].forEach(([dr,dc])=>add(r+dr,c+dc,true));
  }
  if (p.type==="KNIGHT"||p.type==="ASSASSIN") {
    [[-2,-1],[-2,1],[2,-1],[2,1],[-1,-2],[1,-2],[-1,2],[1,2]].forEach(([dr,dc])=>{
      const nr=r+dr,nc=c+dc;
      if(nr<0||nr>=BOARD||nc<0||nc>=BOARD)return;
      if(!board[nr][nc])m.push({row:nr,col:nc});
      else if(board[nr][nc].color!==color)m.push({row:nr,col:nc,attack:true});
    });
    if(p.type==="ASSASSIN") {
      [[-1,0],[1,0],[0,-1],[0,1]].forEach(([dr,dc])=>{
        for(let i=1;i<BOARD-1;i++){
          const er=r+dr*i,ec=c+dc*i;
          if(er<0||er>=BOARD||ec<0||ec>=BOARD)break;
          if(board[er][ec]&&board[er][ec].color!==color){
            const br=er+dr,bc=ec+dc;
            if(br>=0&&br<BOARD&&bc>=0&&bc<BOARD&&!board[br][bc])
              m.push({row:br,col:bc,stealth:true});
            break;
          }
        }
      });
    }
  }
  return m;
};

const getAI = (board) => {
  const all=[];
  for(let r=0;r<BOARD;r++) for(let c=0;c<BOARD;c++){
    const p=board[r][c];
    if(!p||p.color!=="black"||p.stunned>0)continue;
    getMoves(r,c,board,"black").forEach(m=>all.push({
      fromRow:r,fromCol:c,...m,
      score:m.attack?(PIECES_DEF[board[m.row][m.col]?.type]?.hp||0)+100:Math.random()*20
    }));
  }
  if(!all.length)return null;
  all.sort((a,b)=>b.score-a.score);
  return all[Math.floor(Math.random()*Math.min(3,all.length))];
};

const doMove = (board, fR, fC, tR, tC, diceResult) => {
  const nb = board.map(r=>r.map(c=>c?{...c}:null));
  let att = {...nb[fR][fC]};
  const tgt = nb[tR][tC]?{...nb[tR][tC]}:null;
  let log="", dmg=0, fx="move";

  if (tgt) {
    const {dmg:d,fx:f} = diceResult.apply(att.atk||PIECES_DEF[att.type].atk);
    dmg=d; fx=f;
    let nt={...tgt,hp:tgt.hp-dmg};
    if(att.type==="WIZARD")  nt={...nt,burning:true};
    if(att.type==="ARCHER")  nt={...nt,poisoned:true};
    if(att.type==="DRAGON")  nt={...nt,stunned:1};
    if(att.type==="KNIGHT")  att={...att,shielded:true};
    if(att.type==="HEALER")  att={...att,hp:Math.min(att.hp+15,att.maxHp)};
    if(fx==="drain") att={...att,hp:Math.min(att.hp+Math.floor(dmg/2),att.maxHp)};
    log=`${PIECES_DEF[att.type].name} → ${PIECES_DEF[tgt.type].name} [${diceResult.label}] ${dmg}dmg`;
    if(nt.hp<=0){log+=" 💀"; nb[tR][tC]=att;}
    else{nb[tR][tC]=nt; nb[fR][fC]=null; return{board:nb,log,dmg,fx};}
  } else {
    log=`${PIECES_DEF[att.type].name} זז`;
    nb[tR][tC]=att;
  }
  nb[fR][fC]=null;
  return{board:nb,log,dmg,fx};
};

const tickStatus = b => b.map(row=>row.map(p=>{
  if(!p)return null;
  let n={...p};
  if(n.burning){n.hp-=10;n.burning=false;}
  if(n.poisoned)n.hp-=8;
  if(n.stunned>0)n.stunned--;
  if(n.shielded)n.shielded=false;
  return n.hp<=0?null:n;
}));

const checkWin = b => {
  let wd=false,bd=false;
  for(let r=0;r<BOARD;r++)for(let c=0;c<BOARD;c++){
    const p=b[r][c];if(!p)continue;
    if(p.type==="DRAGON"&&p.color==="white")wd=true;
    if(p.type==="DRAGON"&&p.color==="black")bd=true;
  }
  if(!wd)return"שחור";if(!bd)return"לבן";return null;
};

// ═══════════════════════════════════════
// THREE.JS BOARD COMPONENT
// ═══════════════════════════════════════
function ThreeBoard({ board, selected, validMoves, onCellClick, onCellHover, highlightCells }) {
  const mountRef = useRef(null);
  const sceneRef = useRef(null);
  const rendererRef = useRef(null);
  const cameraRef = useRef(null);
  const meshMapRef = useRef({});
  const tileMapRef = useRef({});
  const animFrameRef = useRef(null);
  const clockRef = useRef(new THREE.Clock());

  useEffect(() => {
    const mount = mountRef.current;
    const W = mount.clientWidth, H = mount.clientHeight;

    // Scene
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x0a0612);
    scene.fog = new THREE.FogExp2(0x0a0612, 0.04);
    sceneRef.current = scene;

    // Camera – isometric-style perspective
    const cam = new THREE.PerspectiveCamera(45, W/H, 0.1, 200);
    cam.position.set(0, 22, 18);
    cam.lookAt(0, 0, 0);
    cameraRef.current = cam;

    // Renderer
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(W, H);
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1.2;
    mount.appendChild(renderer.domElement);
    rendererRef.current = renderer;

    // Lights
    const ambient = new THREE.AmbientLight(0x221133, 1.5);
    scene.add(ambient);

    const sunLight = new THREE.DirectionalLight(0xfff5cc, 2.5);
    sunLight.position.set(8, 20, 10);
    sunLight.castShadow = true;
    sunLight.shadow.mapSize.set(2048, 2048);
    sunLight.shadow.camera.near = 0.5;
    sunLight.shadow.camera.far = 100;
    sunLight.shadow.camera.left = -20;
    sunLight.shadow.camera.right = 20;
    sunLight.shadow.camera.top = 20;
    sunLight.shadow.camera.bottom = -20;
    scene.add(sunLight);

    const fillLight = new THREE.PointLight(0x4433ff, 1.5, 40);
    fillLight.position.set(-10, 8, -8);
    scene.add(fillLight);

    const rimLight = new THREE.PointLight(0xff3300, 1.2, 30);
    rimLight.position.set(10, 5, -10);
    scene.add(rimLight);

    // Board base
    const baseGeo = new THREE.BoxGeometry(BOARD*TILE+1, 0.4, BOARD*TILE+1);
    const baseMat = new THREE.MeshStandardMaterial({
      color: 0x1a0d2e, roughness: 0.8, metalness: 0.3,
    });
    const base = new THREE.Mesh(baseGeo, baseMat);
    base.position.y = -0.22;
    base.receiveShadow = true;
    scene.add(base);

    // Edge glow lines
    const edgeGeo = new THREE.EdgesGeometry(baseGeo);
    const edgeMat = new THREE.LineBasicMaterial({ color: 0x6633cc, linewidth: 2 });
    const edges = new THREE.LineSegments(edgeGeo, edgeMat);
    edges.position.copy(base.position);
    scene.add(edges);

    // Tiles
    const offset = ((BOARD - 1) * TILE) / 2;
    for (let r = 0; r < BOARD; r++) {
      for (let c = 0; c < BOARD; c++) {
        const isLight = (r + c) % 2 === 0;
        const geo = new THREE.BoxGeometry(TILE - 0.08, 0.18, TILE - 0.08);
        const mat = new THREE.MeshStandardMaterial({
          color: isLight ? 0x2d1f4e : 0x1a1030,
          roughness: 0.7, metalness: 0.2,
          emissive: isLight ? 0x110820 : 0x080412,
        });
        const tile = new THREE.Mesh(geo, mat);
        tile.position.set(c * TILE - offset, 0, r * TILE - offset);
        tile.receiveShadow = true;
        tile.userData = { row: r, col: c };
        scene.add(tile);
        tileMapRef.current[`${r}-${c}`] = tile;
      }
    }

    // Particle system for atmosphere
    const starGeo = new THREE.BufferGeometry();
    const starVerts = [];
    for (let i = 0; i < 300; i++) {
      starVerts.push(
        (Math.random()-0.5)*60,
        Math.random()*20+2,
        (Math.random()-0.5)*60
      );
    }
    starGeo.setAttribute("position", new THREE.Float32BufferAttribute(starVerts, 3));
    const starMat = new THREE.PointsMaterial({ color: 0xffffff, size: 0.06, transparent: true, opacity: 0.6 });
    scene.add(new THREE.Points(starGeo, starMat));

    // Raycaster for click
    const raycaster = new THREE.Raycaster();
    const mouse = new THREE.Vector2();

    const getCell = (event) => {
      const rect = mount.getBoundingClientRect();
      mouse.x = ((event.clientX - rect.left) / W) * 2 - 1;
      mouse.y = -((event.clientY - rect.top) / H) * 2 + 1;
      raycaster.setFromCamera(mouse, cam);
      const tiles = Object.values(tileMapRef.current);
      const hits = raycaster.intersectObjects(tiles);
      return hits.length ? hits[0].object.userData : null;
    };

    const onClick = e => { const cell = getCell(e); if (cell) onCellClick(cell.row, cell.col); };
    const onHover = e => { const cell = getCell(e); if (cell) onCellHover(cell.row, cell.col); };
    mount.addEventListener("click", onClick);
    mount.addEventListener("mousemove", onHover);

    // Animate
    const animate = () => {
      animFrameRef.current = requestAnimationFrame(animate);
      const t = clockRef.current.getElapsedTime();

      // Floating piece animation
      Object.entries(meshMapRef.current).forEach(([key, mesh]) => {
        if (mesh) {
          mesh.position.y = 0.5 + Math.sin(t * 2 + mesh.userData.phase) * 0.06;
          mesh.rotation.y += 0.008;
        }
      });

      // Pulsing fill light
      fillLight.intensity = 1.5 + Math.sin(t * 1.5) * 0.3;

      renderer.render(scene, cam);
    };
    animate();

    // Resize
    const onResize = () => {
      const W2 = mount.clientWidth, H2 = mount.clientHeight;
      cam.aspect = W2 / H2;
      cam.updateProjectionMatrix();
      renderer.setSize(W2, H2);
    };
    window.addEventListener("resize", onResize);

    return () => {
      cancelAnimationFrame(animFrameRef.current);
      window.removeEventListener("resize", onResize);
      mount.removeEventListener("click", onClick);
      mount.removeEventListener("mousemove", onHover);
      mount.removeChild(renderer.domElement);
      renderer.dispose();
    };
  }, []);

  // Update tiles highlight
  useEffect(() => {
    Object.entries(tileMapRef.current).forEach(([key, tile]) => {
      const [r, c] = key.split("-").map(Number);
      const isSel = selected?.row === r && selected?.col === c;
      const isValid = validMoves.some(m => m.row === r && m.col === c && !m.attack);
      const isAtk = validMoves.some(m => m.row === r && m.col === c && m.attack);
      const isStealth = validMoves.some(m => m.row === r && m.col === c && m.stealth);
      const isLight = (r + c) % 2 === 0;

      tile.material.emissive.set(
        isSel    ? 0xffaa00 :
        isAtk    ? 0x880000 :
        isStealth? 0x550088 :
        isValid  ? 0x005500 :
        isLight  ? 0x110820 : 0x080412
      );
      tile.material.emissiveIntensity = isSel||isAtk||isValid||isStealth ? 0.8 : 0.15;
    });
  }, [selected, validMoves]);

  // Update pieces
  useEffect(() => {
    const scene = sceneRef.current;
    if (!scene) return;
    const offset = ((BOARD - 1) * TILE) / 2;

    // Remove old meshes
    Object.values(meshMapRef.current).forEach(m => { if (m) scene.remove(m); });
    meshMapRef.current = {};

    board.forEach((row, r) => {
      row.forEach((piece, c) => {
        if (!piece) return;
        const def = PIECES_DEF[piece.type];
        const isWhite = piece.color === "white";

        // Base cylinder
        const bodyGeo = new THREE.CylinderGeometry(0.45, 0.55, 0.7, 8);
        const bodyMat = new THREE.MeshStandardMaterial({
          color: isWhite ? 0xe8d5a3 : 0x3d2060,
          emissive: new THREE.Color(def.color).multiplyScalar(0.15),
          roughness: 0.4, metalness: 0.6,
        });
        const body = new THREE.Mesh(bodyGeo, bodyMat);
        body.castShadow = true;

        // Glowing orb on top
        const orbGeo = new THREE.SphereGeometry(0.3, 12, 12);
        const orbMat = new THREE.MeshStandardMaterial({
          color: def.color,
          emissive: def.color,
          emissiveIntensity: 0.8,
          roughness: 0.1, metalness: 0.9,
          transparent: true, opacity: 0.9,
        });
        const orb = new THREE.Mesh(orbGeo, orbMat);
        orb.position.y = 0.65;

        // Point light from orb
        const light = new THREE.PointLight(def.color, 0.8, 3);
        light.position.copy(orb.position);

        // HP bar (scaled box)
        const hpRatio = piece.hp / piece.maxHp;
        const hpGeo = new THREE.BoxGeometry(hpRatio * 0.9, 0.06, 0.1);
        const hpMat = new THREE.MeshStandardMaterial({
          color: hpRatio > 0.6 ? 0x22c55e : hpRatio > 0.3 ? 0xf59e0b : 0xef4444,
          emissive: hpRatio > 0.6 ? 0x22c55e : hpRatio > 0.3 ? 0xf59e0b : 0xef4444,
          emissiveIntensity: 0.5,
        });
        const hpBar = new THREE.Mesh(hpGeo, hpMat);
        hpBar.position.set(-0.45 + hpRatio * 0.45, 1.1, 0);

        // HP bg
        const hpBgGeo = new THREE.BoxGeometry(0.9, 0.06, 0.1);
        const hpBgMat = new THREE.MeshStandardMaterial({ color: 0x333333 });
        const hpBg = new THREE.Mesh(hpBgGeo, hpBgMat);
        hpBg.position.set(0, 1.1, 0);

        // Shield ring
        if (piece.shielded) {
          const ringGeo = new THREE.TorusGeometry(0.65, 0.04, 8, 24);
          const ringMat = new THREE.MeshStandardMaterial({ color:0x60a5fa, emissive:0x60a5fa, emissiveIntensity:1 });
          const ring = new THREE.Mesh(ringGeo, ringMat);
          ring.rotation.x = Math.PI / 2;
          body.add(ring);
        }

        // Group
        const group = new THREE.Group();
        group.add(body);
        group.add(orb);
        group.add(light);
        group.add(hpBg);
        group.add(hpBar);
        group.position.set(c * TILE - offset, 0.5, r * TILE - offset);
        group.userData = { phase: Math.random() * Math.PI * 2 };

        // Status particles
        if (piece.burning || piece.poisoned || piece.stunned > 0) {
          const pGeo = new THREE.SphereGeometry(0.06, 4, 4);
          const pMat = new THREE.MeshStandardMaterial({
            color: piece.burning ? 0xff4400 : piece.poisoned ? 0x8800cc : 0xffff00,
            emissive: piece.burning ? 0xff4400 : piece.poisoned ? 0x8800cc : 0xffff00,
            emissiveIntensity: 1,
          });
          for (let i = 0; i < 4; i++) {
            const pm = new THREE.Mesh(pGeo, pMat);
            const ang = (i / 4) * Math.PI * 2;
            pm.position.set(Math.cos(ang) * 0.5, 0.8 + i * 0.15, Math.sin(ang) * 0.5);
            group.add(pm);
          }
        }

        scene.add(group);
        meshMapRef.current[`${r}-${c}`] = group;
      });
    });
  }, [board]);

  return <div ref={mountRef} style={{ width: "100%", height: "100%", borderRadius: 8 }} />;
}

// ═══════════════════════════════════════
// DICE OVERLAY
// ═══════════════════════════════════════
function DiceOverlay({ rolling, result, onDone }) {
  const [frame, setFrame] = useState(0);
  const faces = ["⚀","⚁","⚂","⚃","⚄","⚅"];

  useEffect(() => {
    if (!rolling) return;
    let n = 0;
    const iv = setInterval(() => {
      setFrame(f => (f+1)%6);
      if (++n >= 20) { clearInterval(iv); setTimeout(onDone, 700); }
    }, 70);
    return () => clearInterval(iv);
  }, [rolling]);

  const show = result || DICE[frame];
  return (
    <div style={{
      position:"absolute", inset:0, background:"rgba(0,0,0,0.7)",
      display:"flex", alignItems:"center", justifyContent:"center",
      zIndex:100, fontFamily:"'Press Start 2P',monospace",
    }}>
      <div style={{
        background:"linear-gradient(135deg,#1a0830,#0d0420)",
        border:`3px solid ${show.color}`,
        borderRadius:16, padding:"28px 44px", textAlign:"center",
        boxShadow:`0 0 50px ${show.color}99, 0 0 100px ${show.color}44`,
        animation: rolling ? "diceShake 0.14s ease-in-out infinite alternate" : "dicePop 0.3s ease-out",
      }}>
        <div style={{
          fontSize:80, lineHeight:1, marginBottom:10,
          filter:`drop-shadow(0 0 16px ${show.color})`,
          animation: rolling ? "diceSpin 0.14s linear infinite" : "none",
        }}>{rolling ? faces[frame] : show.icon}</div>
        <div style={{fontSize:16,color:show.color,letterSpacing:2,marginBottom:6}}>{show.label}</div>
        <div style={{fontSize:9,color:"#aaa"}}>{show.desc}</div>
        {!rolling && result && result.finalDmg > 0 && (
          <div style={{marginTop:12,fontSize:14,color:"#ffd700"}}>💥 {result.finalDmg} נזק</div>
        )}
      </div>
      <style>{`
        @keyframes diceShake{from{transform:translate(-3px,-3px) rotate(-4deg)}to{transform:translate(3px,3px) rotate(4deg)}}
        @keyframes diceSpin{from{transform:rotate(-6deg)}to{transform:rotate(6deg)}}
        @keyframes dicePop{from{transform:scale(0.8);opacity:0}to{transform:scale(1);opacity:1}}
      `}</style>
    </div>
  );
}

// ═══════════════════════════════════════
// MAIN APP
// ═══════════════════════════════════════
export default function LegendChess3D() {
  const [board, setBoard] = useState(initBoard);
  const [selected, setSelected] = useState(null);
  const [validMoves, setValidMoves] = useState([]);
  const [turn, setTurn] = useState("white");
  const [log, setLog] = useState(["⚔️ Legend Chess 3D מוכן!"]);
  const [winner, setWinner] = useState(null);
  const [hoveredCell, setHoveredCell] = useState(null);
  const [diceState, setDiceState] = useState(null);
  const pendingRef = useRef(null);

  const addLog = msg => setLog(p => [msg, ...p.slice(0, 8)]);

  const execMove = (diceResult) => {
    const mv = pendingRef.current;
    if (!mv) return;
    pendingRef.current = null;
    const roll = DICE[Math.floor(Math.random() * 6)];
    const usedRoll = mv.isAttack ? roll : { label:"MOVE", apply:()=>({dmg:0,fx:"move"}) };
    const { board: nb, log: lg, dmg } = doMove(mv.board, mv.fR, mv.fC, mv.tR, mv.tC, usedRoll);
    if (mv.isAttack) {
      setDiceState({ rolling: false, result: { ...roll, finalDmg: dmg } });
      setTimeout(() => {
        const nb2 = tickStatus(nb);
        setBoard(nb2); addLog(lg);
        setDiceState(null);
        const w = checkWin(nb2);
        if (w) { setWinner(w); return; }
        setTurn(mv.nextTurn);
      }, 1400);
    } else {
      const nb2 = tickStatus(nb);
      setBoard(nb2); addLog(lg);
      const w = checkWin(nb2);
      if (w) { setWinner(w); return; }
      setTurn(mv.nextTurn);
    }
  };

  const triggerMove = (board, fR, fC, tR, tC, isAttack, nextTurn) => {
    pendingRef.current = { board, fR, fC, tR, tC, isAttack, nextTurn };
    if (isAttack) setDiceState({ rolling: true, result: null });
    else execMove(null);
  };

  useEffect(() => {
    if (turn !== "black" || winner || diceState) return;
    const t = setTimeout(() => {
      const mv = getAI(board);
      if (!mv) { setTurn("white"); return; }
      triggerMove(board, mv.fromRow, mv.fromCol, mv.toRow, mv.toCol, !!mv.attack, "white");
    }, 800);
    return () => clearTimeout(t);
  }, [turn, board, winner, diceState]);

  const handleCellClick = (r, c) => {
    if (turn !== "white" || winner || diceState) return;
    const piece = board[r][c];
    if (selected) {
      const mv = validMoves.find(m => m.row===r && m.col===c);
      if (mv) {
        triggerMove(board, selected.row, selected.col, r, c, !!mv.attack, "black");
        setSelected(null); setValidMoves([]);
        return;
      }
      if (piece?.color === "white" && !piece.stunned) {
        setSelected({row:r,col:c});
        setValidMoves(getMoves(r,c,board,"white"));
        return;
      }
      setSelected(null); setValidMoves([]);
      return;
    }
    if (piece?.color === "white" && !piece.stunned) {
      setSelected({row:r,col:c});
      setValidMoves(getMoves(r,c,board,"white"));
    }
  };

  const reset = () => {
    setBoard(initBoard()); setSelected(null); setValidMoves([]);
    setTurn("white"); setWinner(null); setDiceState(null);
    pendingRef.current = null;
    setLog(["⚔️ משחק חדש!"]);
  };

  const hovered = hoveredCell ? board[hoveredCell.r]?.[hoveredCell.c] : null;

  return (
    <div style={{
      height:"100vh", width:"100vw", overflow:"hidden",
      background:"#050310",
      display:"flex", flexDirection:"column",
      fontFamily:"'Press Start 2P',monospace", color:"#e2e8f0",
    }}>
      <link href="https://fonts.googleapis.com/css2?family=Press+Start+2P&display=swap" rel="stylesheet"/>

      {/* Header */}
      <div style={{
        display:"flex", alignItems:"center", justifyContent:"space-between",
        padding:"6px 16px",
        background:"linear-gradient(90deg,#0d0420,#1a0830,#0d0420)",
        borderBottom:"1px solid #3d1060",
        flexShrink:0,
      }}>
        <div style={{
          fontSize:13, letterSpacing:3,
          background:"linear-gradient(90deg,#ff4444,#ffd700,#c084fc,#ff4444)",
          backgroundSize:"300%", WebkitBackgroundClip:"text", WebkitTextFillColor:"transparent",
          animation:"shimmer 4s linear infinite",
        }}>⚔ LEGEND CHESS 3D ⚔</div>
        <div style={{fontSize:7, color: turn==="white"?"#ffd700":"#c084fc"}}>
          {winner ? `🏆 ${winner} ניצח!` : turn==="white" ? "[ תורך ]" : "[ AI... ]"}
        </div>
        <button onClick={reset} style={{
          background:"#2d0d4e", color:"#c084fc", border:"1px solid #6633cc",
          padding:"4px 10px", cursor:"pointer", fontSize:7,
        }}>NEW GAME</button>
      </div>

      {/* Main area */}
      <div style={{display:"flex", flex:1, overflow:"hidden", gap:0}}>
        {/* 3D Board */}
        <div style={{flex:1, position:"relative"}}>
          <ThreeBoard
            board={board}
            selected={selected}
            validMoves={validMoves}
            onCellClick={handleCellClick}
            onCellHover={(r,c) => setHoveredCell({r,c})}
          />
          {diceState && (
            <DiceOverlay
              rolling={diceState.rolling}
              result={diceState.rolling ? null : diceState.result}
              onDone={() => { if (diceState.rolling) execMove(null); }}
            />
          )}
        </div>

        {/* Side panel */}
        <div style={{
          width:160, display:"flex", flexDirection:"column", gap:6,
          padding:8, background:"#0a0318",
          borderLeft:"1px solid #2d1050", overflowY:"auto",
        }}>
          {/* Hovered piece */}
          <div style={{background:"#0d0420",border:"1px solid #2d1050",padding:8,minHeight:100}}>
            {hovered ? (
              <>
                <div style={{fontSize:24,marginBottom:4}}>{PIECES_DEF[hovered.type].emoji}</div>
                <div style={{color:PIECES_DEF[hovered.type].color|0,fontSize:7,marginBottom:4}}>
                  {PIECES_DEF[hovered.type].name}
                </div>
                <div style={{fontSize:6,color:"#ef4444"}}>♥ {hovered.hp}/{hovered.maxHp}</div>
                <div style={{fontSize:6,color:"#f59e0b",marginTop:2}}>⚔ {PIECES_DEF[hovered.type].atk}</div>
                <div style={{fontSize:5,color:"#667",marginTop:4,lineHeight:1.6}}>{PIECES_DEF[hovered.type].desc}</div>
                {hovered.burning && <div style={{color:"#f87171",fontSize:5,marginTop:2}}>🔥 בוער</div>}
                {hovered.poisoned && <div style={{color:"#a855f7",fontSize:5}}>☠️ מורעל</div>}
                {hovered.stunned>0 && <div style={{color:"#ffd700",fontSize:5}}>💫 מדוהם</div>}
              </>
            ) : <div style={{color:"#2a1840",fontSize:6,paddingTop:28,textAlign:"center"}}>hover piece</div>}
          </div>

          {/* Dice legend */}
          <div style={{background:"#0d0420",border:"1px solid #2d1050",padding:6}}>
            <div style={{color:"#3d1060",fontSize:5,marginBottom:5}}>🎲 קוביית הגורל</div>
            {DICE.map(d=>(
              <div key={d.label} style={{display:"flex",gap:4,alignItems:"center",marginBottom:3}}>
                <span style={{fontSize:11}}>{d.icon}</span>
                <div>
                  <div style={{color:d.color,fontSize:5}}>{d.label}</div>
                  <div style={{color:"#2a1840",fontSize:4}}>{d.desc}</div>
                </div>
              </div>
            ))}
          </div>

          {/* Battle log */}
          <div style={{background:"#050210",border:"1px solid #1a0830",padding:6,flex:1}}>
            <div style={{color:"#2a1840",fontSize:5,marginBottom:4}}>— LOG —</div>
            {log.map((msg,i)=>(
              <div key={i} style={{
                color:i===0?"#8899bb":"#1a1830",
                fontSize:5,marginBottom:2,lineHeight:1.5,
              }}>{msg}</div>
            ))}
          </div>
        </div>
      </div>

      {winner && (
        <div style={{position:"fixed",inset:0,background:"rgba(0,0,0,0.9)",display:"flex",alignItems:"center",justifyContent:"center",zIndex:999}}>
          <div style={{
            background:"linear-gradient(135deg,#1a0830,#0d0420)",
            border:"3px solid #ffd700",borderRadius:16,
            padding:40,textAlign:"center",
            boxShadow:"0 0 80px rgba(255,215,0,0.5)",
          }}>
            <div style={{fontSize:56,marginBottom:8}}>🏆</div>
            <div style={{fontSize:14,color:"#ffd700",marginBottom:20}}>{winner} ניצח!</div>
            <button onClick={reset} style={{
              background:"linear-gradient(180deg,#ffd700,#cc8800)",
              color:"#000",border:"none",borderRadius:8,
              padding:"12px 24px",cursor:"pointer",fontSize:9,
              boxShadow:"4px 4px 0 #000",
            }}>PLAY AGAIN</button>
          </div>
        </div>
      )}

      <style>{`
        @keyframes shimmer{0%{background-position:0%}100%{background-position:300%}}
        ::-webkit-scrollbar{width:3px}
        ::-webkit-scrollbar-track{background:#050210}
        ::-webkit-scrollbar-thumb{background:#2d1050}
      `}</style>
    </div>
  );
}
