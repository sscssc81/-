import os
import sys
import math
import random
import pygame

# ======================== 常量定义 ========================
BOARD_COLS = 15
BOARD_ROWS = 15
CELL_SIZE  = 38
MARGIN_LEFT = 45
MARGIN_TOP  = 45

WINDOW_WIDTH  = 800
WINDOW_HEIGHT = 600

FRAME_THICKNESS = 24

# 玩家标识
BLACK = 1
WHITE = 2
DRAW  = 3

# AI 难度
DIFFICULTY_EASY   = "easy"
DIFFICULTY_MEDIUM = "medium"
DIFFICULTY_HARD   = "hard"
DIFFICULTY_NAMES  = {DIFFICULTY_EASY: "简 单", DIFFICULTY_MEDIUM: "中 等", DIFFICULTY_HARD: "困 难"}

# 先后手选择
COLOR_CHOICE_BLACK  = "black"
COLOR_CHOICE_WHITE  = "white"
COLOR_CHOICE_RANDOM = "random"

# 交互参数
CLICK_TOLERANCE   = 0.4       # 点击落子容差（相对 CELL_SIZE）
PIECE_RADIUS_RATIO = 0.43     # 棋子半径比例
AI_DELAY_FRAMES    = 25       # AI 思考延迟帧数
HINT_DURATION      = 120      # 提示文字持续帧数

# ════════════════════════════════════════════════════════
#  配色方案 — 暖润柔和的和风木色调
# ════════════════════════════════════════════════════════

# 背景
COLOR_BG = (245, 238, 222)

# 棋盘多层木框（外→内，深→浅）
COLOR_FRAME_OUTER      = (112, 72, 38)
COLOR_FRAME_MID        = (155, 112, 62)
COLOR_FRAME_INNER      = (208, 182, 128)
COLOR_BOARD_SURFACE    = (222, 200, 148)

# 网格线 & 星位
COLOR_LINE   = (72, 52, 30)
COLOR_STAR   = (82, 60, 36)
COLOR_LINE_SUBTLE = (168, 148, 110)         # 微妙的棋盘纹理线（装饰用）

# 棋子
COLOR_BLACK_BASE  = (22, 22, 24)             # 黑子底色
COLOR_BLACK_LOW   = (38, 38, 42)            # 渐变暗部
COLOR_BLACK_MID   = (58, 58, 62)            # 渐变中部
COLOR_BLACK_HI    = (90, 90, 96)            # 高光区
COLOR_BLACK_GLOSS = (130, 130, 138)         # 反光点

COLOR_WHITE_BASE  = (242, 240, 235)         # 白子底色
COLOR_WHITE_LOW   = (195, 190, 182)         # 渐变暗部
COLOR_WHITE_MID   = (225, 222, 215)         # 渐变中部
COLOR_WHITE_HI    = (248, 247, 244)         # 高光区
COLOR_WHITE_GLOSS = (255, 255, 253)         # 反光点

# 棋子阴影
COLOR_SHADOW_SOFT = (155, 140, 120)
COLOR_SHADOW_DARK = (120, 105, 85)

# 文字
COLOR_TEXT        = (58, 40, 22)
COLOR_TEXT_LIGHT  = (120, 92, 64)
COLOR_HIGHLIGHT   = (228, 72, 58)           # 最后落子红点

# 面板
COLOR_PANEL_CARD   = (238, 228, 208)
COLOR_PANEL_BORDER = (175, 145, 105)
COLOR_DIVIDER      = (170, 140, 98)
COLOR_TURN_BLACK_BG = (48, 38, 28)
COLOR_TURN_WHITE_BG  = (230, 222, 206)
COLOR_MODE_TAG     = (140, 115, 78)
COLOR_MODE_TAG_PVE = (90, 125, 148)

# 胜利横幅
COLOR_BANNER_OVERLAY    = (24, 18, 12)
COLOR_BANNER_BLACK_WIN  = (42, 34, 24)
COLOR_BANNER_WHITE_WIN  = (175, 158, 125)
COLOR_BANNER_DRAW       = (115, 105, 88)
COLOR_BANNER_TEXT       = (255, 248, 235)
COLOR_BANNER_TEXT_DARK  = (48, 35, 20)
COLOR_BANNER_BORDER     = (210, 180, 70)
COLOR_BANNER_GLOW       = (255, 215, 60)

# 按钮
COLOR_BTN_RESTART       = (105, 152, 92)
COLOR_BTN_RESTART_HOVER = (136, 182, 122)
COLOR_BTN_UNDO          = (182, 142, 88)
COLOR_BTN_UNDO_HOVER    = (210, 172, 118)
COLOR_BTN_EXIT          = (185, 98, 88)
COLOR_BTN_EXIT_HOVER    = (212, 124, 112)
COLOR_BTN_MODE          = (92, 130, 158)
COLOR_BTN_MODE_HOVER    = (122, 162, 190)
COLOR_BTN_DIFFICULTY    = (130, 115, 155)
COLOR_BTN_DIFFICULTY_H  = (160, 145, 185)
COLOR_BTN_TEXT          = (252, 248, 238)
COLOR_BTN_DISABLED      = (172, 164, 150)
COLOR_BTN_DISABLED_TEXT = (142, 134, 122)
COLOR_BTN_SHADOW        = (70, 52, 30)

# 先后手选择模态框
COLOR_MODAL_OVERLAY     = (18, 14, 10)
COLOR_MODAL_BG          = (240, 232, 210)
COLOR_MODAL_BORDER      = (185, 155, 110)
COLOR_MODAL_OPTION_BG   = (228, 218, 195)
COLOR_MODAL_OPTION_HOV  = (210, 195, 165)
COLOR_MODAL_OPTION_SEL  = (175, 145, 108)
COLOR_MODAL_BTN_CONFIRM = (105, 152, 92)
COLOR_MODAL_BTN_HOVER   = (135, 182, 122)

# 步数统计
COLOR_STEP_BLACK_BG  = (38, 32, 24)
COLOR_STEP_WHITE_BG  = (218, 210, 195)
COLOR_STEP_BLACK_TXT = (225, 218, 205)
COLOR_STEP_WHITE_TXT = (55, 40, 25)

# 动画参数
BANNER_FADE_FRAMES  = 35          # 横幅淡入帧数
BANNER_PULSE_AMOUNT = 0.04        # 脉冲幅度比例

# ======================== 游戏类 ========================
class GobangGame:
    """五子棋游戏主类 — 管理状态、渲染与交互"""

    # 玩家常量别名
    BLACK = BLACK
    WHITE = WHITE
    DRAW  = DRAW

    def __init__(self):
        pygame.init()
        self.screen = pygame.display.set_mode((WINDOW_WIDTH, WINDOW_HEIGHT))
        pygame.display.set_caption("优雅五子棋")

        # 字体（带安全 fallback）
        font_path = self._find_chinese_font()
        self._font_path = font_path  # 可能为 None
        self.font_small   = self._safe_font(font_path, 17)
        self.font         = self._safe_font(font_path, 21)
        self.font_title   = self._safe_font(font_path, 26)
        self.font_banner  = self._safe_font(font_path, 42)
        self.font_victory = self._safe_font(font_path, 52)
        self.font_icon    = self._safe_font(font_path, 36)

        # 游戏状态
        self.board = [[0] * BOARD_COLS for _ in range(BOARD_ROWS)]
        self.current_player = BLACK
        self.game_over      = False
        self.winner         = 0            # 0=进行中, BLACK=黑胜, WHITE=白胜, DRAW=平局
        self.last_move      = None
        self.move_history   = []
        self.move_count     = 0
        self.hover_pos      = None
        self.win_cells      = []
        self.hovered_button = None
        self.hint_message   = ""
        self.hint_timer     = 0

        # 模式 & AI
        self.game_mode    = "pvp"
        self.ai_difficulty = DIFFICULTY_MEDIUM
        self.player_color = COLOR_CHOICE_BLACK   # PVE 模式玩家执黑
        self.ai_thinking  = False
        self.ai_timer     = 0

        # 先后手选择模态框（PVE 模式开始时显示）
        self.show_color_modal  = False
        self.color_modal_hover = None

        # 步数统计（分别记录黑白）
        self.black_moves = 0
        self.white_moves = 0

        # 动画帧计数器
        self.frame       = 0
        self.win_frame   = 0

        # 面板几何
        board_pixel_width = (BOARD_COLS - 1) * CELL_SIZE
        self.panel_x = MARGIN_LEFT + board_pixel_width + FRAME_THICKNESS + 30
        self.buttons = []
        self.clock   = pygame.time.Clock()

        # 预渲染不变的棋盘背景（性能优化）
        self._board_cache = self._render_board_to_surface()

        # 预渲染棋子模板（性能优化）
        self._piece_black = self._render_piece_surface(BLACK)
        self._piece_white = self._render_piece_surface(WHITE)

    # ------------------------------------------------------------
    @staticmethod
    def _safe_font(font_path, size):
        """安全创建字体：文件路径失败时回退到系统默认"""
        if font_path is None:
            return pygame.font.Font(None, size)
        try:
            return pygame.font.Font(font_path, size)
        except (IOError, OSError, pygame.error):
            return pygame.font.Font(None, size)

    # ------------------------------------------------------------
    @staticmethod
    def _find_chinese_font():
        """多策略查找系统中文字体，始终返回可用路径或 None（由 _safe_font 兜底）"""
        # 策略1：直接文件路径
        win_dir = os.path.join(os.environ.get("SystemRoot", "C:\\Windows"), "Fonts")
        candidates = [
            os.path.join(win_dir, "simhei.ttf"),
            os.path.join(win_dir, "msyh.ttc"),
            os.path.join(win_dir, "msyhbd.ttc"),
            os.path.join(win_dir, "simsun.ttc"),
            os.path.join(win_dir, "simkai.ttf"),
            os.path.join(win_dir, "simfang.ttf"),
        ]
        for p in candidates:
            if os.path.isfile(p):
                return p

        # 策略2：通过 pygame 系统字体匹配
        for name in ["simhei", "microsoft yahei", "simsun", "simkai"]:
            try:
                sf = pygame.font.SysFont(name, 22)
                if sf.render("测", True, (0, 0, 0)).get_width() > 10:
                    fp = pygame.font.match_font(name)
                    if fp:
                        return fp
            except Exception:
                continue

        # 策略3：尝试常见跨平台中文字体
        cross_platform = [
            "/System/Library/Fonts/PingFang.ttc",          # macOS
            "/System/Library/Fonts/STHeiti Light.ttc",     # macOS
            "/usr/share/fonts/truetype/wqy/wqy-zenhei.ttc",# Linux
            "/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc", # Linux
        ]
        for p in cross_platform:
            if os.path.isfile(p):
                return p

        return None

    # ============================================================
    #  按钮系统
    # ============================================================
    def _build_buttons(self, frame_x, frame_y, frame_w, frame_h):
        """在外框内构建按钮矩形（5个按钮均匀分布）"""
        btn_w = 120
        btn_h = 34
        btn_count = 5
        gap = 8
        total_h = btn_h * btn_count + gap * (btn_count - 1)
        start_y = frame_y + (frame_h - total_h) // 2
        btn_x = frame_x + (frame_w - btn_w) // 2

        mode_text = "人机对战" if self.game_mode == "pvp" else "双人对战"
        diff_text = DIFFICULTY_NAMES.get(self.ai_difficulty, "中 等")

        definitions = [
            ("restart",    "再来一局", COLOR_BTN_RESTART, COLOR_BTN_RESTART_HOVER),
            ("undo",       "悔    棋", COLOR_BTN_UNDO,    COLOR_BTN_UNDO_HOVER),
            ("mode",       mode_text,  COLOR_BTN_MODE,    COLOR_BTN_MODE_HOVER),
            ("difficulty", diff_text,  COLOR_BTN_DIFFICULTY, COLOR_BTN_DIFFICULTY_H),
            ("exit",       "退出游戏", COLOR_BTN_EXIT,     COLOR_BTN_EXIT_HOVER),
        ]

        self.buttons = []
        for idx, (bid, text, base, hover) in enumerate(definitions):
            rect = pygame.Rect(btn_x, start_y + idx * (btn_h + gap), btn_w, btn_h)
            self.buttons.append({
                "id": bid, "rect": rect, "text": text,
                "base_color": base, "hover_color": hover,
            })

    def _draw_buttons(self):
        """绘制按钮（带柔和阴影与圆角）"""
        for btn in self.buttons:
            bid  = btn["id"]
            rect = btn["rect"]
            text = btn["text"]
            disabled = (bid == "undo" and (self.move_count == 0 or self.game_over))

            if disabled:
                bg = COLOR_BTN_DISABLED
                txt_color = COLOR_BTN_DISABLED_TEXT
            elif self.hovered_button == bid:
                bg = btn["hover_color"]
                txt_color = COLOR_BTN_TEXT
                # 悬停时微放大效果
                inflate = 2
                rect = pygame.Rect(rect.x - inflate, rect.y - inflate,
                                   rect.width + inflate * 2, rect.height + inflate * 2)
            else:
                bg = btn["base_color"]
                txt_color = COLOR_BTN_TEXT

            r = 10  # 圆角半径

            # 柔和阴影
            sr = pygame.Rect(rect.x + 2, rect.y + 2, rect.width, rect.height)
            pygame.draw.rect(self.screen, COLOR_BTN_SHADOW, sr, border_radius=r)

            # 按钮主体
            pygame.draw.rect(self.screen, bg, rect, border_radius=r)

            # 顶部微亮边缘（模拟光线）
            highlight_rect = pygame.Rect(rect.x + 2, rect.y + 1, rect.width - 4, rect.height // 2)
            hl_surf = pygame.Surface((highlight_rect.width, highlight_rect.height), pygame.SRCALPHA)
            hl_surf.fill((255, 255, 255, 18))
            self.screen.blit(hl_surf, (highlight_rect.x, highlight_rect.y))

            # 文字居中
            txt_surf = self.font_small.render(text, True, txt_color)
            tx = rect.x + (rect.width - txt_surf.get_width()) // 2
            ty = rect.y + (rect.height - txt_surf.get_height()) // 2
            self.screen.blit(txt_surf, (tx, ty))

    def _handle_button_click(self, mouse_pos):
        mx, my = mouse_pos
        for btn in self.buttons:
            if btn["rect"].collidepoint(mx, my):
                bid = btn["id"]
                if bid == "restart":     self._restart()
                elif bid == "undo":      self._undo()
                elif bid == "mode":      self._toggle_mode()
                elif bid == "difficulty": self._toggle_difficulty()
                elif bid == "exit":      self._exit()
                return True
        return False

    def _toggle_mode(self):
        self.game_mode = "pve" if self.game_mode == "pvp" else "pvp"
        self._restart()

    def _toggle_difficulty(self):
        order = [DIFFICULTY_EASY, DIFFICULTY_MEDIUM, DIFFICULTY_HARD]
        idx = order.index(self.ai_difficulty)
        self.ai_difficulty = order[(idx + 1) % len(order)]
        self._show_hint(f"AI 难度已切换为：{DIFFICULTY_NAMES[self.ai_difficulty]}")
        self._restart()

    def _restart(self):
        self.board = [[0] * BOARD_COLS for _ in range(BOARD_ROWS)]
        self.current_player = BLACK
        self.game_over      = False
        self.winner         = 0
        self.last_move      = None
        self.move_history   = []
        self.move_count     = 0
        self.hover_pos      = None
        self.win_cells      = []
        self.hovered_button = None
        self.ai_thinking    = False
        self.ai_timer       = 0
        self.hint_message   = ""
        self.hint_timer     = 0
        self.win_frame      = 0
        self.black_moves    = 0
        self.white_moves    = 0
        # PVE 模式显示先后手选择模态框
        if self.game_mode == "pve":
            self.show_color_modal = True
            self.color_modal_hover = None
        else:
            self.show_color_modal = False

    def _undo(self):
        if self.game_over:
            self._show_hint("游戏已结束，无法悔棋"); return
        if self.ai_thinking:
            self._show_hint("AI 正在思考，请稍候"); return
        if self.move_count == 0:
            self._show_hint("当前没有棋子可悔"); return

        if self.game_mode == "pve":
            if self.move_count < 2:
                self._show_hint("当前没有棋子可悔"); return
            # 回退 AI 一步
            r_ai, c_ai = self.move_history.pop()
            ai_piece = self.board[r_ai][c_ai]
            self.board[r_ai][c_ai] = 0
            self.move_count -= 1
            self._decrement_step(ai_piece)
            # 回退人类一步
            r_pl, c_pl = self.move_history.pop()
            hu_piece = self.board[r_pl][c_pl]
            self.board[r_pl][c_pl] = 0
            self.move_count -= 1
            self._decrement_step(hu_piece)
            self.current_player = self._human_color
        else:
            r, c = self.move_history.pop()
            piece = self.board[r][c]
            self.board[r][c] = 0
            self._decrement_step(piece)
            self.current_player = WHITE if self.current_player == BLACK else BLACK
            self.move_count -= 1

        self.last_move  = self.move_history[-1] if self.move_history else None
        self.win_cells   = []
        self.hover_pos   = None
        self.ai_thinking = False
        self.ai_timer    = 0
        self.win_frame   = 0

    def _decrement_step(self, piece):
        """根据棋子颜色减少步数计数"""
        if piece == BLACK:
            self.black_moves = max(0, self.black_moves - 1)
        elif piece == WHITE:
            self.white_moves = max(0, self.white_moves - 1)

    def _show_hint(self, message):
        self.hint_message = message
        self.hint_timer   = HINT_DURATION

    def _trigger_ai(self):
        self.ai_thinking = True
        self.ai_timer    = AI_DELAY_FRAMES

    @staticmethod
    def _exit():
        pygame.quit()
        sys.exit()

    # ============================================================
    #  先后手选择模态框（PVE 模式）
    # ============================================================
    def _update_color_modal_hover(self, mouse_pos):
        if not self.show_color_modal:
            self.color_modal_hover = None
            return
        mx, my = mouse_pos
        opts = self._color_modal_options()
        self.color_modal_hover = None
        for key, rect in opts:
            if rect.collidepoint(mx, my):
                self.color_modal_hover = key
                break

    def _handle_color_modal_click(self, mouse_pos):
        if not self.show_color_modal:
            return False
        mx, my = mouse_pos
        opts = self._color_modal_options()
        for key, rect in opts:
            if rect.collidepoint(mx, my):
                self._select_color(key)
                return True
        return False

    def _select_color(self, choice):
        if choice == COLOR_CHOICE_RANDOM:
            choice = random.choice([COLOR_CHOICE_BLACK, COLOR_CHOICE_WHITE])
        self.player_color = choice
        self.show_color_modal = False
        self.color_modal_hover = None
        # 确定先后手：如果玩家执黑，玩家先手；否则 AI（执黑）先手
        if self.player_color == COLOR_CHOICE_BLACK:
            self.current_player = BLACK
            self._show_hint("您执黑先行")
        else:
            self.current_player = BLACK   # AI 先手
            self._trigger_ai()

    def _color_modal_options(self):
        """返回模态框选项的 (key, rect) 列表"""
        w, h = 380, 260
        bx = (WINDOW_WIDTH - w) // 2
        by = (WINDOW_HEIGHT - h) // 2 - 30

        opt_w, opt_h = 90, 100
        gap = 24
        total_w = opt_w * 3 + gap * 2
        start_x = bx + (w - total_w) // 2
        opt_y = by + 90

        options = [
            (COLOR_CHOICE_BLACK,  "●", "执黑先行"),
            (COLOR_CHOICE_WHITE,  "○", "执白后手"),
            (COLOR_CHOICE_RANDOM, "?", "随机分配"),
        ]
        result = []
        for i, (key, icon, label) in enumerate(options):
            ox = start_x + i * (opt_w + gap)
            result.append((key, pygame.Rect(ox, opt_y, opt_w, opt_h)))
        return result

    def _draw_color_modal(self):
        """绘制先后手选择模态框"""
        if not self.show_color_modal:
            return

        # 半透明遮罩
        overlay = pygame.Surface((WINDOW_WIDTH, WINDOW_HEIGHT), pygame.SRCALPHA)
        overlay.fill((*COLOR_MODAL_OVERLAY, 168))
        self.screen.blit(overlay, (0, 0))

        w, h = 380, 260
        bx = (WINDOW_WIDTH - w) // 2
        by = (WINDOW_HEIGHT - h) // 2 - 30

        # 对话框背景
        bg_rect = pygame.Rect(bx, by, w, h)
        pygame.draw.rect(self.screen, COLOR_MODAL_BG, bg_rect, border_radius=16)
        pygame.draw.rect(self.screen, COLOR_MODAL_BORDER, bg_rect, 2, border_radius=16)

        # 标题
        title = self.font_title.render("选择先后手", True, COLOR_TEXT)
        self.screen.blit(title, (bx + (w - title.get_width()) // 2, by + 22))

        # 装饰线
        lx, ly = bx + 30, by + 60
        pygame.draw.line(self.screen, COLOR_DIVIDER, (lx, ly), (bx + w - 30, ly), 1)

        # 三个选项
        options_data = [
            (COLOR_CHOICE_BLACK,  "●", "执黑先行"),
            (COLOR_CHOICE_WHITE,  "○", "执白后手"),
            (COLOR_CHOICE_RANDOM, "?", "随机分配"),
        ]
        opt_w, opt_h = 90, 100
        gap = 24
        total_w = opt_w * 3 + gap * 2
        start_x = bx + (w - total_w) // 2
        opt_y = by + 72

        for i, (key, icon, label) in enumerate(options_data):
            ox = start_x + i * (opt_w + gap)
            orect = pygame.Rect(ox, opt_y, opt_w, opt_h)

            # 选项背景
            if self.color_modal_hover == key:
                opt_bg = COLOR_MODAL_OPTION_HOV
                border_color = COLOR_MODAL_BORDER
            elif self.player_color == key:
                opt_bg = COLOR_MODAL_OPTION_SEL
                border_color = (160, 125, 80)
            else:
                opt_bg = COLOR_MODAL_OPTION_BG
                border_color = (200, 185, 155)

            pygame.draw.rect(self.screen, opt_bg, orect, border_radius=12)
            pygame.draw.rect(self.screen, border_color, orect, 2, border_radius=12)

            # 图标（大号棋子符号）
            if key == COLOR_CHOICE_RANDOM:
                icon_color = (135, 105, 55)
            elif key == COLOR_CHOICE_BLACK:
                icon_color = (28, 26, 22)
            else:
                icon_color = (220, 215, 205)
            icon_surf = self.font_icon.render(icon, True, icon_color)
            self.screen.blit(icon_surf, (ox + (opt_w - icon_surf.get_width()) // 2,
                                         opt_y + 12))
            # 标签
            lbl_surf = self.font_small.render(label, True, COLOR_TEXT)
            self.screen.blit(lbl_surf, (ox + (opt_w - lbl_surf.get_width()) // 2,
                                        opt_y + opt_h - 26))

    # ============================================================
    #  AI 算法（简单 / 中等 / 困难）
    # ============================================================
    @property
    def _ai_color(self):
        """返回 AI 的棋子颜色"""
        return WHITE if self.player_color == COLOR_CHOICE_BLACK else BLACK

    @property
    def _human_color(self):
        """返回人类玩家的棋子颜色"""
        return BLACK if self.player_color == COLOR_CHOICE_BLACK else WHITE

    def _ai_move(self):
        """根据难度分发 AI 决策"""
        if self.ai_difficulty == DIFFICULTY_EASY:
            self._ai_move_easy()
        elif self.ai_difficulty == DIFFICULTY_MEDIUM:
            self._ai_move_medium()
        elif self.ai_difficulty == DIFFICULTY_HARD:
            self._ai_move_hard()

    # ── 简单 AI：随机落子 ──
    def _ai_move_easy(self):
        empty = [(r, c) for r in range(BOARD_ROWS) for c in range(BOARD_COLS)
                 if self.board[r][c] == 0]
        if empty:
            r, c = random.choice(empty)
            self._apply_ai_move(r, c)

    # ── 中等 AI：防守优先（原算法） ──
    def _ai_move_medium(self):
        ai_c = self._ai_color
        hu_c = self._human_color

        # ① AI 立即获胜
        for r in range(BOARD_ROWS):
            for c in range(BOARD_COLS):
                if self.board[r][c] != 0: continue
                self.board[r][c] = ai_c
                if self._check_win(r, c):
                    self.board[r][c] = 0
                    self._apply_ai_move(r, c); return
                self.board[r][c] = 0

        # ② 堵截玩家
        for r in range(BOARD_ROWS):
            for c in range(BOARD_COLS):
                if self.board[r][c] != 0: continue
                self.board[r][c] = hu_c
                if self._check_win(r, c):
                    self.board[r][c] = 0
                    self._apply_ai_move(r, c); return
                self.board[r][c] = 0

        # ③ 评分选优
        self._ai_evaluate_and_move(ai_c, hu_c, attack_weight=1.0, defense_weight=1.1)

    # ── 困难 AI：进攻优先 ──
    def _ai_move_hard(self):
        ai_c = self._ai_color
        hu_c = self._human_color

        # ① AI 立即获胜
        for r in range(BOARD_ROWS):
            for c in range(BOARD_COLS):
                if self.board[r][c] != 0: continue
                self.board[r][c] = ai_c
                if self._check_win(r, c):
                    self.board[r][c] = 0
                    self._apply_ai_move(r, c); return
                self.board[r][c] = 0

        # ② 堵截玩家
        for r in range(BOARD_ROWS):
            for c in range(BOARD_COLS):
                if self.board[r][c] != 0: continue
                self.board[r][c] = hu_c
                if self._check_win(r, c):
                    self.board[r][c] = 0
                    self._apply_ai_move(r, c); return
                self.board[r][c] = 0

        # ③ 评分选优 — 进攻权重更高
        self._ai_evaluate_and_move(ai_c, hu_c, attack_weight=1.2, defense_weight=1.0)

    def _ai_evaluate_and_move(self, ai_color, human_color, attack_weight, defense_weight):
        """评估所有空位并按综合分选最优"""
        candidates = []
        for r in range(BOARD_ROWS):
            for c in range(BOARD_COLS):
                if self.board[r][c] != 0: continue
                score = self._eval_position_weighted(r, c, ai_color, human_color,
                                                     attack_weight, defense_weight)
                candidates.append((score, r, c))

        candidates.sort(key=lambda x: x[0], reverse=True)
        best_score = candidates[0][0]
        top = [(r, c) for s, r, c in candidates if s == best_score]
        best_move = random.choice(top)
        self._apply_ai_move(*best_move)

    def _apply_ai_move(self, row, col):
        ai_c = self._ai_color
        self.board[row][col] = ai_c
        self.last_move = (row, col)
        self.move_history.append((row, col))
        self.move_count += 1
        if ai_c == BLACK:
            self.black_moves += 1
        else:
            self.white_moves += 1

        win_line = self._check_win(row, col)
        if win_line:
            self.game_over, self.winner, self.win_cells = True, ai_c, win_line
            self.win_frame = self.frame
        elif self.move_count >= BOARD_COLS * BOARD_ROWS:
            self.game_over, self.winner = True, DRAW
            self.win_frame = self.frame
        else:
            self.current_player = self._human_color

    def _eval_position_weighted(self, row, col, ai_color, human_color, att_w, def_w):
        """按权重评估位置的进攻/防守分"""
        directions = [(0, 1), (1, 0), (1, 1), (1, -1)]
        attack = sum(self._eval_line(row, col, dr, dc, ai_color) for dr, dc in directions)
        defense = sum(self._eval_line(row, col, dr, dc, human_color) for dr, dc in directions)
        return int(attack * att_w + defense * def_w)

    def _eval_line(self, row, col, dr, dc, player):
        count, open_ends = 1, 0
        r, c = row + dr, col + dc
        while 0 <= r < BOARD_ROWS and 0 <= c < BOARD_COLS:
            if self.board[r][c] == player:    count += 1
            elif self.board[r][c] == 0:       open_ends += 1; break
            else:                             break
            r += dr; c += dc
        r, c = row - dr, col - dc
        while 0 <= r < BOARD_ROWS and 0 <= c < BOARD_COLS:
            if self.board[r][c] == player:    count += 1
            elif self.board[r][c] == 0:       open_ends += 1; break
            else:                             break
            r -= dr; c -= dc
        if count >= 5:   return 100000
        if count == 4:   return 50000 if open_ends == 2 else 5000
        if count == 3:   return 5000  if open_ends == 2 else 500
        if count == 2:   return 500   if open_ends == 2 else 50
        if count == 1:   return 50    if open_ends == 2 else 5
        return 0

    # ============================================================
    #  交互
    # ============================================================
    @staticmethod
    def _screen_to_board(mouse_pos):
        """将鼠标像素坐标转换为棋盘 (row, col)，失败返回 None"""
        mx, my = mouse_pos
        col = round((mx - MARGIN_LEFT) / CELL_SIZE)
        row = round((my - MARGIN_TOP) / CELL_SIZE)
        if not (0 <= col < BOARD_COLS and 0 <= row < BOARD_ROWS):
            return None
        cx, cy = MARGIN_LEFT + col * CELL_SIZE, MARGIN_TOP + row * CELL_SIZE
        if abs(mx - cx) > CELL_SIZE * CLICK_TOLERANCE or abs(my - cy) > CELL_SIZE * CLICK_TOLERANCE:
            return None
        return (row, col)

    def _update_hover(self, mouse_pos):
        if self.show_color_modal:
            self.hover_pos = None
            return
        pos = self._screen_to_board(mouse_pos)
        if pos is None:
            self.hover_pos = None
            return
        r, c = pos
        self.hover_pos = None if self.board[r][c] != 0 else (r, c)

    def _update_button_hover(self, mouse_pos):
        mx, my = mouse_pos
        self.hovered_button = None
        for btn in self.buttons:
            if btn["rect"].collidepoint(mx, my):
                bid = btn["id"]
                if bid == "undo" and (self.move_count == 0 or self.game_over):
                    continue
                self.hovered_button = bid
                break

    def run(self):
        running = True
        while running:
            for event in pygame.event.get():
                if event.type == pygame.QUIT:
                    running = False
                elif event.type == pygame.MOUSEMOTION:
                    self._update_hover(event.pos)
                    self._update_button_hover(event.pos)
                    self._update_color_modal_hover(event.pos)
                elif event.type == pygame.MOUSEBUTTONDOWN:
                    # 颜色选择模态框优先处理
                    if self.show_color_modal:
                        if self._handle_color_modal_click(event.pos):
                            continue
                    if self._handle_button_click(event.pos):
                        continue
                    if not self.game_over and not self.show_color_modal:
                        self._handle_click(event.pos)

            if self.ai_thinking and not self.game_over:
                if self.ai_timer > 0:
                    self.ai_timer -= 1
                else:
                    self._ai_move()
                    self.ai_thinking = False

            self._draw()
            self.frame += 1
            self.clock.tick(60)

        pygame.quit()
        sys.exit()

    # ------------------------------------------------------------
    def _handle_click(self, mouse_pos):
        # PVE 模式：仅人类回合可落子
        if self.game_mode == "pve":
            if self.current_player != self._human_color or self.ai_thinking:
                return
        # 颜色选择模态框打开时不处理棋盘点击
        if self.show_color_modal:
            return

        pos = self._screen_to_board(mouse_pos)
        if pos is None:
            return
        row, col = pos
        if self.board[row][col] != 0:
            return

        self.board[row][col] = self.current_player
        self.last_move = (row, col)
        self.move_history.append((row, col))
        self.move_count += 1
        if self.current_player == BLACK:
            self.black_moves += 1
        else:
            self.white_moves += 1

        win_line = self._check_win(row, col)
        if win_line:
            self.game_over = True
            self.winner = self.current_player
            self.win_cells = win_line
            self.win_frame = self.frame
        elif self.move_count >= BOARD_COLS * BOARD_ROWS:
            self.game_over = True
            self.winner = DRAW
            self.win_frame = self.frame
        else:
            self.current_player = WHITE if self.current_player == BLACK else BLACK
            if self.game_mode == "pve" and self.current_player == self._ai_color:
                self._trigger_ai()

    # ------------------------------------------------------------
    def _check_win(self, row, col):
        player = self.board[row][col]
        directions = [(0, 1), (1, 0), (1, 1), (1, -1)]
        for dr, dc in directions:
            cells = [(row, col)]
            r, c = row + dr, col + dc
            while 0 <= r < BOARD_ROWS and 0 <= c < BOARD_COLS and self.board[r][c] == player:
                cells.append((r, c)); r += dr; c += dc
            r, c = row - dr, col - dc
            while 0 <= r < BOARD_ROWS and 0 <= c < BOARD_COLS and self.board[r][c] == player:
                cells.append((r, c)); r -= dr; c -= dc
            if len(cells) >= 5:
                cells.sort()
                return cells[:5]
        return None

    # ============================================================
    #  绘制系统
    # ============================================================
    def _draw(self):
        self.screen.fill(COLOR_BG)

        # 棋盘背景（预渲染，直接 blit）
        self.screen.blit(self._board_cache, (0, 0))

        # 绘制所有棋子（使用预渲染缓存）
        for r in range(BOARD_ROWS):
            for c in range(BOARD_COLS):
                if self.board[r][c] != 0:
                    self._draw_piece_cached(r, c)

        # 最后落子标记（脉动红点）
        if self.last_move and not self.game_over:
            r, c = self.last_move
            cx = MARGIN_LEFT + c * CELL_SIZE
            cy = MARGIN_TOP + r * CELL_SIZE
            pulse = 1.0 + 0.15 * math.sin(self.frame * 0.08)
            dot_r = int(5 * pulse)
            pygame.draw.circle(self.screen, COLOR_HIGHLIGHT, (cx, cy), dot_r)

        # 获胜连线高亮 — 双层金色光晕 + 脉动动画
        if self.game_over and self.win_cells:
            elapsed = self.frame - self.win_frame
            pulse = 1.0 + 0.12 * math.sin(elapsed * 0.06)
            for r, c in self.win_cells:
                cx = MARGIN_LEFT + c * CELL_SIZE
                cy = MARGIN_TOP + r * CELL_SIZE
                # 内层粗光环
                ring_r = int(CELL_SIZE * 0.46 * pulse)
                alpha = 220
                ring_surf = pygame.Surface((ring_r * 2 + 4, ring_r * 2 + 4), pygame.SRCALPHA)
                pygame.draw.circle(ring_surf, (255, 200, 40, alpha),
                                   (ring_r + 2, ring_r + 2), ring_r, 3)
                self.screen.blit(ring_surf, (cx - ring_r - 2, cy - ring_r - 2))

                # 外层半透明光晕
                glow_r = int(CELL_SIZE * PIECE_RADIUS_RATIO * pulse)
                glow_alpha = int(80 + 30 * math.sin(elapsed * 0.04))
                glow_surf = pygame.Surface((glow_r * 2 + 2, glow_r * 2 + 2), pygame.SRCALPHA)
                pygame.draw.circle(glow_surf, (255, 220, 80, glow_alpha),
                                   (glow_r + 1, glow_r + 1), glow_r)
                self.screen.blit(glow_surf, (cx - glow_r - 1, cy - glow_r - 1))

        self._draw_hover_preview()
        self._draw_info_panel()

        if self.game_over:
            self._draw_winner_banner()

        # 先后手选择模态框（最顶层）
        if self.show_color_modal:
            self._draw_color_modal()

        if self.hint_timer > 0:
            self.hint_timer -= 1

        pygame.display.flip()

    # ------------------------------------------------------------
    def _render_board_to_surface(self):
        """预渲染棋盘到独立 Surface，每帧直接 blit 即可"""
        surf = pygame.Surface((WINDOW_WIDTH, WINDOW_HEIGHT))
        surf.fill(COLOR_BG)

        board_w = (BOARD_COLS - 1) * CELL_SIZE
        board_h = (BOARD_ROWS - 1) * CELL_SIZE
        ox, oy = MARGIN_LEFT, MARGIN_TOP
        ft = FRAME_THICKNESS

        ol = ox - ft; ot = oy - ft; ow = board_w + ft * 2; oh = board_h + ft * 2

        # ① 外层阴影
        shadow_rect = pygame.Rect(ol + 3, ot + 3, ow + 2, oh + 2)
        pygame.draw.rect(surf, (78, 58, 32), shadow_rect, border_radius=8)

        # ② 外框
        outer = pygame.Rect(ol, ot, ow, oh)
        pygame.draw.rect(surf, COLOR_FRAME_OUTER, outer, border_radius=8)

        # ③ 中框
        mid = pygame.Rect(ol + 4, ot + 4, ow - 8, oh - 8)
        pygame.draw.rect(surf, COLOR_FRAME_MID, mid, border_radius=5)

        # ④ 内框
        inner = pygame.Rect(ol + 10, ot + 10, ow - 20, oh - 20)
        pygame.draw.rect(surf, COLOR_FRAME_INNER, inner, border_radius=4)

        # ⑤ 棋盘面板
        board_rect = pygame.Rect(ol + 13, ot + 13, ow - 26, oh - 26)
        pygame.draw.rect(surf, COLOR_BOARD_SURFACE, board_rect)

        # ⑥ 细微纹理线
        for i in range(3):
            ly = board_rect.y + 6 + i * (board_rect.height - 12) // 2
            pygame.draw.line(surf, COLOR_LINE_SUBTLE, (board_rect.x + 10, ly),
                             (board_rect.right - 10, ly), 1)

        # ⑦ 网格线
        for i in range(BOARD_ROWS):
            y = oy + i * CELL_SIZE
            pygame.draw.line(surf, COLOR_LINE, (ox, y), (ox + board_w, y), 1)
        for j in range(BOARD_COLS):
            x = ox + j * CELL_SIZE
            pygame.draw.line(surf, COLOR_LINE, (x, oy), (x, oy + board_h), 1)

        # ⑧ 星位
        stars = [(3, 3), (3, 7), (3, 11), (7, 3), (7, 7), (7, 11),
                 (11, 3), (11, 7), (11, 11)]
        for r, c in stars:
            sx, sy = ox + c * CELL_SIZE, oy + r * CELL_SIZE
            pygame.draw.circle(surf, COLOR_STAR, (sx, sy), 3)

        return surf

    # ------------------------------------------------------------
    def _render_piece_surface(self, player):
        """预渲染一颗棋子到带透明的 Surface（与棋盘尺寸无关的模板）"""
        R = int(CELL_SIZE * PIECE_RADIUS_RATIO)
        size = R * 2 + 8  # 包含阴影边距
        surf = pygame.Surface((size, size), pygame.SRCALPHA)
        center = size // 2

        is_black = (player == BLACK)

        if is_black:
            layers = [
                (COLOR_BLACK_BASE,  R),
                (COLOR_BLACK_LOW,   int(R * 0.82)),
                (COLOR_BLACK_MID,   int(R * 0.62)),
                (COLOR_BLACK_HI,    int(R * 0.40)),
            ]
            rim_color  = (50, 50, 54)
            gloss_color = COLOR_BLACK_GLOSS
        else:
            layers = [
                (COLOR_WHITE_BASE,  R),
                (COLOR_WHITE_LOW,   int(R * 0.82)),
                (COLOR_WHITE_MID,   int(R * 0.62)),
                (COLOR_WHITE_HI,    int(R * 0.40)),
            ]
            rim_color  = (148, 142, 132)
            gloss_color = COLOR_WHITE_GLOSS

        # ① 软阴影
        shadow_r = R + 2
        pygame.draw.circle(surf, (*COLOR_SHADOW_SOFT, 80),
                           (center + 2, center + 2), shadow_r)

        # ② 硬阴影
        pygame.draw.circle(surf, (*COLOR_SHADOW_DARK, 100),
                           (center + 1, center + 1), R)

        # ③ 逐层渐变
        for color, radius in layers:
            pygame.draw.circle(surf, color, (center, center), radius)

        # ④ 高光反光点
        gloss_off = int(R * 0.32)
        gloss_r   = max(2, int(R * 0.22))
        pygame.draw.circle(surf, (*gloss_color, 200),
                           (center - gloss_off, center - gloss_off), gloss_r)

        # ⑤ 白子外圈细边
        if not is_black:
            pygame.draw.circle(surf, rim_color, (center, center), R, 1)

        return surf

    # ------------------------------------------------------------
    def _draw_piece_cached(self, row, col):
        """使用预渲染模板快速绘制棋子"""
        cx = MARGIN_LEFT + col * CELL_SIZE
        cy = MARGIN_TOP + row * CELL_SIZE
        template = self._piece_black if self.board[row][col] == BLACK else self._piece_white
        size = template.get_width()
        self.screen.blit(template, (cx - size // 2, cy - size // 2))

    # ------------------------------------------------------------
    def _draw_hover_preview(self):
        if self.hover_pos is None or self.game_over:
            return
        if self.show_color_modal:
            return
        if self.game_mode == "pve" and (self.current_player != self._human_color or self.ai_thinking):
            return
        r, c = self.hover_pos
        cx = MARGIN_LEFT + c * CELL_SIZE
        cy = MARGIN_TOP + r * CELL_SIZE
        R = int(CELL_SIZE * PIECE_RADIUS_RATIO)

        preview = pygame.Surface((R * 2, R * 2), pygame.SRCALPHA)
        alpha = 80
        if self.current_player == BLACK:
            color = (*COLOR_BLACK_BASE, alpha)
        else:
            color = (*COLOR_WHITE_BASE, alpha)
        pygame.draw.circle(preview, color, (R, R), R)
        self.screen.blit(preview, (cx - R, cy - R))

    # ------------------------------------------------------------
    def _draw_winner_banner(self):
        """
        带动画的胜利横幅：淡入 + 脉冲缩放 + 金色光晕呼吸。
        """
        elapsed  = self.frame - self.win_frame

        # 淡入进度 (0→1)
        fade_progress = min(1.0, elapsed / BANNER_FADE_FRAMES)
        # 缓出函数 (ease-out)
        fade = 1.0 - (1.0 - fade_progress) ** 3

        # 脉冲缩放
        pulse = 1.0 + BANNER_PULSE_AMOUNT * math.sin(elapsed * 0.05)

        # 半透明暗色蒙版
        overlay_alpha = int(110 * fade)
        overlay = pygame.Surface((WINDOW_WIDTH, WINDOW_HEIGHT), pygame.SRCALPHA)
        overlay.fill((*COLOR_BANNER_OVERLAY, overlay_alpha))
        self.screen.blit(overlay, (0, 0))

        # 文字与配色
        if self.winner == BLACK:
            line1, icon = "黑 棋 胜 利", "●"
            bg_color = COLOR_BANNER_BLACK_WIN
            text_color = COLOR_BANNER_TEXT
            sub_color  = (195, 178, 150)
        elif self.winner == WHITE:
            line1, icon = "白 棋 胜 利", "○"
            bg_color = COLOR_BANNER_WHITE_WIN
            text_color = COLOR_BANNER_TEXT_DARK
            sub_color  = (110, 82, 52)
        else:
            line1, icon = "平    局", "— ◆ —"
            bg_color = COLOR_BANNER_DRAW
            text_color = COLOR_BANNER_TEXT
            sub_color  = (200, 190, 170)

        # 渲染文字
        surf1 = self.font_victory.render(line1, True, text_color)
        surf_icon = self.font_icon.render(icon, True, sub_color)
        w1, h1 = surf1.get_width(), surf1.get_height()
        wi, hi = surf_icon.get_width(), surf_icon.get_height()

        total_h = h1 + 6 + hi
        pad_x, pad_y = 48, 24
        bw = max(w1, wi) + pad_x * 2
        bh = total_h + pad_y * 2

        # 脉冲缩放后的尺寸
        bw_p = int(bw * pulse)
        bh_p = int(bh * pulse)

        bx = (WINDOW_WIDTH - bw_p) // 2
        by = (WINDOW_HEIGHT - bh_p) // 2 - 20

        # 光晕（金色脉动圆角矩形）
        glow_alpha = int(50 * fade * (0.7 + 0.3 * math.sin(elapsed * 0.06)))
        glow_r = pygame.Rect(bx - 8, by - 8, bw_p + 16, bh_p + 16)
        gs = pygame.Surface((glow_r.width, glow_r.height), pygame.SRCALPHA)
        gs.fill((*COLOR_BANNER_GLOW, glow_alpha))
        self.screen.blit(gs, (glow_r.x, glow_r.y))

        # 横幅背景 (带淡入透明度)
        bg_alpha = int(255 * fade)
        bs = pygame.Surface((bw_p, bh_p), pygame.SRCALPHA)
        bs.fill((*bg_color, bg_alpha))
        self.screen.blit(bs, (bx, by))

        # 金边
        border_alpha = int(255 * fade)
        pygame.draw.rect(self.screen, (*COLOR_BANNER_BORDER, border_alpha),
                         (bx, by, bw_p, bh_p), 3, border_radius=14)

        # 内嵌装饰线
        inner_alpha = int(100 * fade)
        inner_rect = pygame.Rect(bx + 8, by + 8, bw_p - 16, bh_p - 16)
        pygame.draw.rect(self.screen, (*COLOR_BANNER_BORDER, inner_alpha),
                         inner_rect, 1, border_radius=10)

        # 文字（带淡入）
        text_alpha = int(255 * min(1.0, fade * 1.3))
        if text_alpha > 0:
            s1_copy = surf1.copy()
            s1_copy.set_alpha(text_alpha)
            si_copy = surf_icon.copy()
            si_copy.set_alpha(text_alpha)

            tx1 = bx + (bw_p - w1) // 2
            ty1 = by + pad_y + int((bh_p - pad_y * 2 - total_h) * 0.4)
            self.screen.blit(s1_copy, (tx1, ty1))

            tx2 = bx + (bw_p - wi) // 2
            ty2 = ty1 + h1 + 6
            self.screen.blit(si_copy, (tx2, ty2))

    # ------------------------------------------------------------
    def _draw_info_panel(self):
        """绘制右侧信息面板 — 与棋盘边框上下对齐"""
        px = self.panel_x
        card_w = 160

        # ── 面板边界：与棋盘外框上下对齐 ──
        ft = FRAME_THICKNESS
        panel_top    = MARGIN_TOP - ft                         # 21（棋盘外框顶部）
        panel_bottom = MARGIN_TOP + (BOARD_ROWS - 1) * CELL_SIZE + ft  # 601（棋盘外框底部）

        def _card(y, h, border=False):
            """绘制圆角卡片，带微阴影"""
            rect = pygame.Rect(px, y, card_w, h)
            # 卡片阴影
            sr = pygame.Rect(px + 1, y + 1, card_w, h)
            pygame.draw.rect(self.screen, (190, 176, 150), sr, border_radius=8)
            pygame.draw.rect(self.screen, COLOR_PANEL_CARD, rect, border_radius=8)
            if border:
                pygame.draw.rect(self.screen, COLOR_PANEL_BORDER, rect, 1, border_radius=8)
            return rect

        # ══════════════ 标题 ══════════════
        y = panel_top + 12                                       # 面板顶部留 12px
        title = self.font_title.render("优雅五子棋", True, COLOR_TEXT)
        self.screen.blit(title, (px + 6, y))
        y += title.get_height() + 10

        # ══════════════ 模式标签 ══════════════
        mode_text = "双人对战" if self.game_mode == "pvp" else "人机对战"
        mode_surf = self.font_small.render(mode_text, True, (248, 242, 228))
        tw, th = mode_surf.get_size()
        tag_bg = COLOR_MODE_TAG if self.game_mode == "pvp" else COLOR_MODE_TAG_PVE
        tag_rect = pygame.Rect(px + 4, y, tw + 16, th + 6)
        pygame.draw.rect(self.screen, tag_bg, tag_rect, border_radius=10)
        self.screen.blit(mode_surf, (px + 12, y + 3))

        if self.game_mode == "pve":
            # 难度标签（紧跟模式标签右侧）
            diff_text = DIFFICULTY_NAMES[self.ai_difficulty]
            diff_surf = self.font_small.render(diff_text, True, (248, 242, 228))
            diff_bg = pygame.Rect(px + 4 + tag_rect.width + 6, y,
                                  diff_surf.get_width() + 14, th + 6)
            pygame.draw.rect(self.screen, COLOR_BTN_DIFFICULTY, diff_bg, border_radius=10)
            self.screen.blit(diff_surf, (diff_bg.x + 7, y + 3))
        y += tag_rect.height + 14

        # ══════════════ 分割线 ══════════════
        pygame.draw.line(self.screen, COLOR_DIVIDER, (px, y), (px + card_w, y), 1)
        y += 14

        # ══════════════ 回合 / 结果卡片 ══════════════
        CARD_TURN_H = 64
        if not self.game_over:
            _card(y, CARD_TURN_H, border=True)

            # 标签行
            label = self.font_small.render("当前回合", True, COLOR_TEXT_LIGHT)
            self.screen.blit(label, (px + 10, y + 6))

            # 棋子文字标签 — 与右侧圆点居中对齐
            if self.current_player == BLACK:
                turn_str, turn_color = "黑棋", (240, 235, 225)
                turn_bg = COLOR_TURN_BLACK_BG
            else:
                turn_str, turn_color = "白棋", (55, 40, 25)
                turn_bg = COLOR_TURN_WHITE_BG

            t_surf = self.font.render(turn_str, True, turn_color)
            tw2, th2 = t_surf.get_size()
            text_label_x = px + 10
            text_label_y = y + 24
            t_rect = pygame.Rect(text_label_x, text_label_y, tw2 + 18, th2 + 6)
            pygame.draw.rect(self.screen, turn_bg, t_rect, border_radius=6)
            if self.current_player == WHITE:
                pygame.draw.rect(self.screen, (155, 128, 85), t_rect, 1, border_radius=6)
            self.screen.blit(t_surf, (text_label_x + 9, text_label_y + 3))

            # PVE 模式：显示操作者标签
            if self.game_mode == "pve":
                if self.ai_thinking:
                    role_txt = "AI 思考中..."
                    role_color = (148, 115, 72)
                elif self.current_player == self._human_color:
                    role_txt = "（你的回合）"
                    role_color = (85, 140, 85)
                else:
                    role_txt = "（AI 回合）"
                    role_color = (138, 108, 70)
                role_surf = self.font_small.render(role_txt, True, role_color)
                role_y = text_label_y + t_rect.height + 2
                self.screen.blit(role_surf, (px + 12, role_y))

            # ── 右侧指示圆点（与文字行垂直居中，完全在卡片内）──
            dot_r = 7                                              # 缩小半径防溢出
            dot_cx = px + card_w - 14                              # 距右边缘 14px
            dot_cy = text_label_y + th2 // 2 + 4                   # 与文字标签行对齐
            # 用 clip 限制绘制范围防阴影溢出
            clip_rect = pygame.Rect(px + 1, y + 1, card_w - 2, CARD_TURN_H - 2)
            self.screen.set_clip(clip_rect)
            dot_color = COLOR_BLACK_BASE if self.current_player == BLACK else COLOR_WHITE_BASE
            pygame.draw.circle(self.screen, COLOR_SHADOW_SOFT,
                               (dot_cx + 1, dot_cy + 1), dot_r)
            pygame.draw.circle(self.screen, dot_color, (dot_cx, dot_cy), dot_r)
            if self.current_player == WHITE:
                pygame.draw.circle(self.screen, (150, 145, 135), (dot_cx, dot_cy), dot_r, 1)
            self.screen.set_clip(None)

            y += CARD_TURN_H + 8
        else:
            _card(y, CARD_TURN_H, border=True)
            label = self.font_small.render("对局结果", True, COLOR_TEXT_LIGHT)
            self.screen.blit(label, (px + 10, y + 6))
            if self.winner == BLACK:
                res, rc = "黑棋胜利", (38, 32, 28)
            elif self.winner == WHITE:
                res, rc = "白棋胜利", (88, 72, 52)
            else:
                res, rc = "平    局", (135, 105, 55)
            r_surf = self.font.render(res, True, rc)
            self.screen.blit(r_surf, (px + 10, y + 28))
            y += CARD_TURN_H + 8

        # ══════════════ 落子计数（增强版：黑/白分别统计） ══════════════
        CARD_COUNT_H = 58
        _card(y, CARD_COUNT_H)
        lbl = self.font_small.render("步数统计", True, COLOR_TEXT_LIGHT)
        self.screen.blit(lbl, (px + 10, y + 4))

        # 黑棋 / 白棋 分列统计
        stat_y = y + 24
        half_w = card_w // 2
        for i, (color_tag, cnt_val, bg, txt_c, border_c) in enumerate([
            ("黑", self.black_moves, COLOR_STEP_BLACK_BG, COLOR_STEP_BLACK_TXT, (90, 82, 68)),
            ("白", self.white_moves, COLOR_STEP_WHITE_BG, COLOR_STEP_WHITE_TXT, (155, 128, 85)),
        ]):
            sx = px + 8 + i * half_w
            s_rect = pygame.Rect(sx, stat_y, half_w - 10, 24)
            pygame.draw.rect(self.screen, bg, s_rect, border_radius=5)
            if i == 1:
                pygame.draw.rect(self.screen, border_c, s_rect, 1, border_radius=5)
            s_txt = f"{color_tag}: {cnt_val}"
            s_surf = self.font_small.render(s_txt, True, txt_c)
            self.screen.blit(s_surf, (sx + 6, stat_y + 3))
        y += CARD_COUNT_H + 8

        # ══════════════ 按钮区（填充到面板底部） ══════════════
        # 与棋盘外框底部对齐，上下留相同边距
        y += 2
        panel_pad = 12                                             # 面板底部留白
        frame_h = panel_bottom - panel_pad - y                     # 动态高度到棋盘底
        frame_rect = pygame.Rect(px, y, card_w, frame_h)

        # 按钮区外框阴影
        fr_sr = pygame.Rect(px + 2, y + 2, card_w, frame_h)
        pygame.draw.rect(self.screen, COLOR_BTN_SHADOW, fr_sr, border_radius=8)
        pygame.draw.rect(self.screen, COLOR_DIVIDER, frame_rect, 1, border_radius=8)

        self._build_buttons(px, y, card_w, frame_h)
        self._draw_buttons()

        # ══════════════ 提示文字 ══════════════
        if self.hint_timer > 0:
            hint_y = y + frame_h + 6
            h_surf = self.font_small.render(self.hint_message, True, COLOR_HIGHLIGHT)
            tw_h = h_surf.get_width()
            self.screen.blit(h_surf, (px + (card_w - tw_h) // 2, hint_y))

# ======================== 入口 ========================
if __name__ == "__main__":
    game = GobangGame()
    game.run()

