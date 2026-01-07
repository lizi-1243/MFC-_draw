# AI 使用日志（项目二：简易绘图板(GDI + 撤销/重做)）

## 对话记录详情
### 2025.12.9 10:15 - 11:30
#### Prompt
我正在基于 MFC + GDI 开发简易绘图板，需要实现画布初始化功能，要求主窗口包含绘图区（使用 CStatic 或自定义 CView），通过双缓冲防止闪烁，并且要应用 Strategy 设计模式中的 IRenderStrategy 接口。请提供该功能的核心代码框架，包括 IRenderStrategy 接口定义、双缓冲实现逻辑以及绘图区与主窗口的绑定方式，同时说明关键代码的作用。

#### Response
```cpp
// IRenderStrategy 接口定义（Strategy 设计模式）
class IRenderStrategy {
public:
    virtual ~IRenderStrategy() = default;
    // 渲染接口，参数为绘图DC和画布区域
    virtual void Render(CDC* pDC, const CRect& rect) = 0;
};

// 双缓冲渲染策略实现
class DoubleBufferRenderStrategy : public IRenderStrategy {
public:
    void Render(CDC* pDC, const CRect& rect) override {
        // 创建内存DC
        CDC memDC;
        memDC.CreateCompatibleDC(pDC);
        // 创建兼容位图
        CBitmap memBmp;
        memBmp.CreateCompatibleBitmap(pDC, rect.Width(), rect.Height());
        // 保存旧位图
        CBitmap* pOldBmp = memDC.SelectObject(&memBmp);
        
        // 填充背景（白色）
        memDC.FillSolidRect(rect, RGB(255, 255, 255));
        
        // 此处可添加具体图形绘制逻辑（后续由业务层扩展）
        // 示例：绘制边框
        memDC.Draw3dRect(rect, RGB(0, 0, 0), RGB(128, 128, 128));
        
        // 双缓冲关键：将内存DC内容拷贝到目标DC
        pDC->BitBlt(rect.left, rect.top, rect.Width(), rect.Height(), 
                    &memDC, 0, 0, SRCCOPY);
        
        // 恢复旧位图，避免资源泄漏（RAII思想）
        memDC.SelectObject(pOldBmp);
    }
};

// 自定义绘图视图类（继承 CView）
class CDrawingView : public CView {
    DECLARE_DYNCREATE(CDrawingView)
private:
    std::unique_ptr<IRenderStrategy> m_pRenderStrategy; // 渲染策略对象
    CRect m_drawRect; // 绘图区域
    
public:
    CDrawingView() {
        // 默认使用双缓冲渲染策略
        m_pRenderStrategy = std::make_unique<DoubleBufferRenderStrategy>();
    }
    
    // 设置渲染策略（Strategy 模式切换能力）
    void SetRenderStrategy(std::unique_ptr<IRenderStrategy> pStrategy) {
        if (pStrategy) {
            m_pRenderStrategy = std::move(pStrategy);
        }
    }
    
protected:
    // 绘图消息响应
    afx_msg void OnDraw(CDC* pDC) override {
        CView::OnDraw(pDC);
        // 获取绘图区域
        GetClientRect(&m_drawRect);
        // 调用策略接口进行渲染
        m_pRenderStrategy->Render(pDC, m_drawRect);
    }
    
    DECLARE_MESSAGE_MAP()
};

BEGIN_MESSAGE_MAP(CDrawingView, CView)
END_MESSAGE_MAP()

// 主窗口（CMainFrame）中绑定绘图视图
BOOL CMainFrame::OnCreateClient(LPCREATESTRUCT lpcs, CCreateContext* pContext) {
    // 创建绘图视图
    if (!m_wndSplitter.CreateStatic(this, 1, 1)) {
        return FALSE;
    }
    if (!m_wndSplitter.CreateView(0, 0, RUNTIME_CLASS(CDrawingView), CSize(800, 600), pContext)) {
        m_wndSplitter.DestroyWindow();
        return FALSE;
    }
    // 设置主视图
    SetActiveView((CView*)m_wndSplitter.GetPane(0, 0));
    return TRUE;
}
```

关键说明：
1. IRenderStrategy 接口定义了统一的渲染行为，后续可扩展其他渲染策略（如不同背景色、不同抗锯齿方式），符合 Strategy 模式的开闭原则。
2. DoubleBufferRenderStrategy 实现了双缓冲逻辑，通过内存 DC 预绘制内容，再一次性拷贝到目标 DC，彻底解决绘图闪烁问题。
3. CDrawingView 作为自定义绘图视图，持有渲染策略对象，通过 SetRenderStrategy 方法可动态切换渲染方式。
4. 主窗口通过 CSplitterWnd 绑定绘图视图，确保绘图区在主窗口中正常显示，符合 MFC 窗口架构规范。

#### Diff（代码修改与整合）
1. 在项目头文件中添加了 IRenderStrategy 接口和 DoubleBufferRenderStrategy 类的声明，补充了必要的头文件引用（#include <memory>、#include <afxext.h>）。
2. 对 CDrawingView 类进行了扩展，添加了成员变量 m_drawRect 用于存储绘图区域，在 OnDraw 方法中补充了视图刷新时的区域更新逻辑。
3. 调整了 CMainFrame 中 OnCreateClient 方法的窗口大小参数，将默认画布尺寸改为 1000x800，更符合实际使用场景。
4. 添加了 RAII 风格的资源管理逻辑，确保 memDC 和 memBmp 在使用后正确释放，避免 GDI 资源泄漏。

### 2025.12.12 14:30 - 16:00
#### Prompt
现在需要实现绘图板的基本图形绘制功能，包括直线、矩形、椭圆、自由曲线，要求通过鼠标左键开始绘制、移动时预览、松开时完成绘制，并且应用 Factory 设计模式（ShapeFactory）来创建不同图形实例。请提供图形基类、具体图形类、ShapeFactory 工厂类的核心代码，以及鼠标消息响应的逻辑，确保与之前的双缓冲渲染策略兼容。

#### Response
```cpp
// 图形基类（抽象产品）
class IShape {
public:
    virtual ~IShape() = default;
    // 设置图形属性（颜色、线宽）
    virtual void SetAttribute(COLORREF color, int lineWidth) = 0;
    // 设置图形顶点（起点、终点、临时点）
    virtual void SetPoints(const CPoint& start, const CPoint& end, const CPoint& temp = CPoint()) = 0;
    // 绘制图形（支持预览和最终绘制）
    virtual void Draw(CDC* pDC) = 0;
    // 获取图形类型
    virtual CString GetShapeType() const = 0;
};

// 直线类（具体产品）
class CLineShape : public IShape {
private:
    CPoint m_startPoint;
    CPoint m_endPoint;
    COLORREF m_color;
    int m_lineWidth;
public:
    void SetAttribute(COLORREF color, int lineWidth) override {
        m_color = color;
        m_lineWidth = lineWidth;
    }
    
    void SetPoints(const CPoint& start, const CPoint& end, const CPoint& temp) override {
        m_startPoint = start;
        m_endPoint = end;
    }
    
    void Draw(CDC* pDC) override {
        CPen pen(PS_SOLID, m_lineWidth, m_color);
        CPen* pOldPen = pDC->SelectObject(&pen);
        pDC->MoveTo(m_startPoint);
        pDC->LineTo(m_endPoint);
        pDC->SelectObject(pOldPen);
    }
    
    CString GetShapeType() const override { return _T("Line"); }
};

// 矩形类（具体产品）
class CRectangleShape : public IShape {
private:
    CPoint m_startPoint;
    CPoint m_endPoint;
    COLORREF m_color;
    int m_lineWidth;
public:
    void SetAttribute(COLORREF color, int lineWidth) override {
        m_color = color;
        m_lineWidth = lineWidth;
    }
    
    void SetPoints(const CPoint& start, const CPoint& end, const CPoint& temp) override {
        m_startPoint = start;
        m_endPoint = end;
    }
    
    void Draw(CDC* pDC) override {
        CPen pen(PS_SOLID, m_lineWidth, m_color);
        CPen* pOldPen = pDC->SelectObject(&pen);
        CBrush* pOldBrush = (CBrush*)pDC->SelectStockObject(NULL_BRUSH); // 透明画刷
        pDC->Rectangle(CRect(m_startPoint, m_endPoint));
        pDC->SelectObject(pOldPen);
        pDC->SelectObject(pOldBrush);
    }
    
    CString GetShapeType() const override { return _T("Rectangle"); }
};

// 椭圆类（具体产品）
class CEllipseShape : public IShape {
private:
    CPoint m_startPoint;
    CPoint m_endPoint;
    COLORREF m_color;
    int m_lineWidth;
public:
    void SetAttribute(COLORREF color, int lineWidth) override {
        m_color = color;
        m_lineWidth = lineWidth;
    }
    
    void SetPoints(const CPoint& start, const CPoint& end, const CPoint& temp) override {
        m_startPoint = start;
        m_endPoint = end;
    }
    
    void Draw(CDC* pDC) override {
        CPen pen(PS_SOLID, m_lineWidth, m_color);
        CPen* pOldPen = pDC->SelectObject(&pen);
        CBrush* pOldBrush = (CBrush*)pDC->SelectStockObject(NULL_BRUSH);
        pDC->Ellipse(CRect(m_startPoint, m_endPoint));
        pDC->SelectObject(pOldPen);
        pDC->SelectObject(pOldBrush);
    }
    
    CString GetShapeType() const override { return _T("Ellipse"); }
};

// 自由曲线类（具体产品）
class CFreeCurveShape : public IShape {
private:
    std::vector<CPoint> m_points;
    COLORREF m_color;
    int m_lineWidth;
public:
    void SetAttribute(COLORREF color, int lineWidth) override {
        m_color = color;
        m_lineWidth = lineWidth;
    }
    
    void SetPoints(const CPoint& start, const CPoint& end, const CPoint& temp) override {
        if (m_points.empty()) {
            m_points.push_back(start);
        }
        m_points.push_back(temp); // 移动时添加临时点
    }
    
    void Draw(CDC* pDC) override {
        if (m_points.size() < 2) return;
        CPen pen(PS_SOLID, m_lineWidth, m_color);
        CPen* pOldPen = pDC->SelectObject(&pen);
        pDC->MoveTo(m_points[0]);
        for (size_t i = 1; i < m_points.size(); ++i) {
            pDC->LineTo(m_points[i]);
        }
        pDC->SelectObject(pOldPen);
    }
    
    CString GetShapeType() const override { return _T("FreeCurve"); }
};

// 图形工厂类（工厂方法模式）
class ShapeFactory {
public:
    enum ShapeType {
        ST_LINE,
        ST_RECTANGLE,
        ST_ELLIPSE,
        ST_FREE_CURVE
    };
    
    // 静态工厂方法，根据类型创建图形实例
    static std::unique_ptr<IShape> CreateShape(ShapeType type) {
        switch (type) {
            case ST_LINE:
                return std::make_unique<CLineShape>();
            case ST_RECTANGLE:
                return std::make_unique<CRectangleShape>();
            case ST_ELLIPSE:
                return std::make_unique<CEllipseShape>();
            case ST_FREE_CURVE:
                return std::make_unique<CFreeCurveShape>();
            default:
                throw std::invalid_argument("Invalid shape type");
        }
    }
};

// 绘图视图类中添加鼠标消息响应和图形管理逻辑
class CDrawingView : public CView {
    // 原有代码省略...
private:
    ShapeFactory::ShapeType m_currentShapeType; // 当前选中的图形类型
    std::unique_ptr<IShape> m_pCurrentShape; // 当前正在绘制的图形
    std::vector<std::unique_ptr<IShape>> m_shapes; // 已完成绘制的图形集合
    CPoint m_startPoint; // 鼠标起点
    COLORREF m_currentColor; // 当前颜色
    int m_currentLineWidth; // 当前线宽
    
public:
    // 设置当前图形类型
    void SetCurrentShapeType(ShapeFactory::ShapeType type) {
        m_currentShapeType = type;
    }
    
    // 设置当前绘图属性
    void SetCurrentAttribute(COLORREF color, int lineWidth) {
        m_currentColor = color;
        m_currentLineWidth = lineWidth;
    }
    
protected:
    // 鼠标左键按下消息
    afx_msg void OnLButtonDown(UINT nFlags, CPoint point) {
        CView::OnLButtonDown(nFlags, point);
        // 创建当前类型的图形（工厂模式）
        m_pCurrentShape = ShapeFactory::CreateShape(m_currentShapeType);
        if (m_pCurrentShape) {
            // 设置图形属性
            m_pCurrentShape->SetAttribute(m_currentColor, m_currentLineWidth);
            // 设置起点
            m_startPoint = point;
            m_pCurrentShape->SetPoints(m_startPoint, point);
        }
        SetCapture(); // 捕获鼠标
    }
    
    // 鼠标移动消息
    afx_msg void OnMouseMove(UINT nFlags, CPoint point) {
        CView::OnMouseMove(nFlags, point);
        if (m_pCurrentShape && (nFlags & MK_LBUTTON)) {
            // 双缓冲预览：先刷新背景，再绘制已完成图形和当前预览图形
            CDC* pDC = GetDC();
            CRect rect;
            GetClientRect(&rect);
            
            // 调用双缓冲渲染策略绘制背景
            m_pRenderStrategy->Render(pDC, rect);
            
            // 绘制已完成的图形
            for (const auto& pShape : m_shapes) {
                pShape->Draw(pDC);
            }
            
            // 更新当前图形的终点/临时点并绘制预览
            m_pCurrentShape->SetPoints(m_startPoint, point, point);
            m_pCurrentShape->Draw(pDC);
            
            ReleaseDC(pDC);
        }
    }
    
    // 鼠标左键松开消息
    afx_msg void OnLButtonUp(UINT nFlags, CPoint point) {
        CView::OnLButtonUp(nFlags, point);
        if (m_pCurrentShape) {
            // 确认当前图形的终点
            m_pCurrentShape->SetPoints(m_startPoint, point);
            // 将完成的图形添加到集合中
            m_shapes.push_back(std::move(m_pCurrentShape));
            // 刷新视图，绘制最终图形
            InvalidateRect(NULL, FALSE);
        }
        ReleaseCapture(); // 释放鼠标
    }
    
    // 重写 OnDraw 方法，绘制所有已完成图形
    afx_msg void OnDraw(CDC* pDC) override {
        CView::OnDraw(pDC);
        GetClientRect(&m_drawRect);
        // 双缓冲绘制背景
        m_pRenderStrategy->Render(pDC, m_drawRect);
        // 绘制所有已完成的图形
        for (const auto& pShape : m_shapes) {
            pShape->Draw(pDC);
        }
    }
    
    DECLARE_MESSAGE_MAP()
};

BEGIN_MESSAGE_MAP(CDrawingView, CView)
    ON_WM_LBUTTONDOWN()
    ON_WM_MOUSEMOVE()
    ON_WM_LBUTTONUP()
END_MESSAGE_MAP()
```

关键说明：
1. IShape 作为图形基类，定义了统一的属性设置、顶点设置和绘制接口，所有具体图形类均继承自该接口，符合里氏替换原则。
2. ShapeFactory 工厂类通过静态方法 CreateShape 根据图形类型创建具体实例，将图形创建与使用解耦，后续添加新图形（如三角形）时无需修改使用端代码，符合开闭原则。
3. 鼠标消息响应逻辑中，OnLButtonDown 捕获起点并创建图形实例，OnMouseMove 实时更新图形顶点并通过双缓冲预览，OnLButtonUp 确认图形并添加到集合，流程清晰且符合用户操作习惯。
4. 所有图形绘制均基于 CDC 实现，与之前的双缓冲渲染策略完全兼容，确保绘图无闪烁。

#### Diff（代码修改与整合）
1. 完善了 IShape 基类的接口设计，添加了纯虚析构函数，避免析构时的资源泄漏问题。
2. 在 CFreeCurveShape 类中优化了临时点的存储逻辑，添加了清空点集合的 Reset 方法，方便重复绘制自由曲线。
3. 对 CDrawingView 类的鼠标消息响应代码进行了调整，在 OnMouseMove 中添加了边界检测，防止图形绘制超出画布范围。
4. 补充了图形属性的默认值设置（默认颜色为黑色，线宽为 2），在 CDrawingView 的构造函数中初始化 m_currentColor 和 m_currentLineWidth。
5. 添加了 GDI 资源释放的严格检查，确保 CPen、CBrush 等资源在使用后正确恢复，符合 RAII 设计思想。

### 2025.12.15 09:00 - 10:45
#### Prompt
需要实现撤销/重做功能，要求每次完成操作生成 Command 对象，使用 Undo/Redo 栈来管理。请提供 Command 接口、具体的图形操作 Command 类、Undo/Redo 栈的管理逻辑，并且整合到之前的绘图板代码中，确保与现有功能（图形绘制、双缓冲）兼容，同时说明 Command 设计模式的应用方式。

#### Response
```cpp
// Command 接口（命令模式）
class ICommand {
public:
    virtual ~ICommand() = default;
    // 执行命令
    virtual void Execute() = 0;
    // 撤销命令
    virtual void Undo() = 0;
    // 重做命令
    virtual void Redo() = 0;
    // 获取命令描述
    virtual CString GetDescription() const = 0;
};

// 绘制图形命令（具体命令）
class DrawShapeCommand : public ICommand {
private:
    CDrawingView* m_pDrawingView; // 关联的绘图视图
    std::unique_ptr<IShape> m_pShape; // 命令对应的图形
    size_t m_shapeIndex; // 图形在集合中的索引（用于撤销时删除）
    
public:
    // 构造函数：传入视图和图形
    DrawShapeCommand(CDrawingView* pView, std::unique_ptr<IShape> pShape)
        : m_pDrawingView(pView), m_pShape(std::move(pShape)) {}
    
    void Execute() override {
        // 执行绘制：将图形添加到视图的图形集合
        if (m_pDrawingView && m_pShape) {
            m_shapeIndex = m_pDrawingView->m_shapes.size();
            m_pDrawingView->m_shapes.push_back(std::move(m_pShape));
            m_pDrawingView->InvalidateRect(NULL, FALSE); // 刷新视图
        }
    }
    
    void Undo() override {
        // 撤销：从图形集合中移除该图形
        if (m_pDrawingView && m_shapeIndex < m_pDrawingView->m_shapes.size()) {
            // 恢复图形所有权，用于重做
            m_pShape = std::move(m_pDrawingView->m_shapes[m_shapeIndex]);
            // 从集合中删除
            m_pDrawingView->m_shapes.erase(m_pDrawingView->m_shapes.begin() + m_shapeIndex);
            m_pDrawingView->InvalidateRect(NULL, FALSE);
        }
    }
    
    void Redo() override {
        // 重做：重新添加图形到集合
        if (m_pDrawingView && m_pShape) {
            m_pDrawingView->m_shapes.insert(m_pDrawingView->m_shapes.begin() + m_shapeIndex, 
                                           std::move(m_pShape));
            m_pDrawingView->InvalidateRect(NULL, FALSE);
        }
    }
    
    CString GetDescription() const override {
        if (m_pShape) {
            return _T("绘制图形：") + m_pShape->GetShapeType();
        }
        return _T("绘制图形（未知类型）");
    }
};

// 绘图视图类中添加 Undo/Redo 栈管理
class CDrawingView : public CView {
    // 原有代码省略...
public:
    std::stack<std::unique_ptr<ICommand>> m_undoStack; // 撤销栈
    std::stack<std::unique_ptr<ICommand>> m_redoStack; // 重做栈
    
    // 执行命令并压入撤销栈
    void ExecuteCommand(std::unique_ptr<ICommand> pCommand) {
        if (pCommand) {
            pCommand->Execute();
            m_undoStack.push(std::move(pCommand));
            // 执行新命令后，清空重做栈（符合操作习惯）
            while (!m_redoStack.empty()) {
                m_redoStack.pop();
            }
        }
    }
    
    // 撤销操作
    void Undo() {
        if (!m_undoStack.empty()) {
            auto pCommand = std::move(m_undoStack.top());
            m_undoStack.pop();
            pCommand->Undo();
            m_redoStack.push(std::move(pCommand));
        }
    }
    
    // 重做操作
    void Redo() {
        if (!m_redoStack.empty()) {
            auto pCommand = std::move(m_redoStack.top());
            m_redoStack.pop();
            pCommand->Redo();
            m_undoStack.push(std::move(pCommand));
        }
    }
    
    // 检查是否可撤销
    bool CanUndo() const { return !m_undoStack.empty(); }
    // 检查是否可重做
    bool CanRedo() const { return !m_redoStack.empty(); }
};

// 主窗口添加快捷键响应（Ctrl+Z 撤销，Ctrl+Y 重做）
BEGIN_MESSAGE_MAP(CMainFrame, CFrameWndEx)
    // 原有消息映射省略...
    ON_COMMAND(ID_EDIT_UNDO, &CMainFrame::OnEditUndo)
    ON_COMMAND(ID_EDIT_REDO, &CMainFrame::OnEditRedo)
    ON_UPDATE_COMMAND_UI(ID_EDIT_UNDO, &CMainFrame::OnUpdateEditUndo)
    ON_UPDATE_COMMAND_UI(ID_EDIT_REDO, &CMainFrame::OnUpdateEditRedo)
END_MESSAGE_MAP()

void CMainFrame::OnEditUndo() {
    CDrawingView* pView = dynamic_cast<CDrawingView*>(GetActiveView());
    if (pView) {
        pView->Undo();
    }
}

void CMainFrame::OnEditRedo() {
    CDrawingView* pView = dynamic_cast<CDrawingView*>(GetActiveView());
    if (pView) {
        pView->Redo();
    }
}

void CMainFrame::OnUpdateEditUndo(CCmdUI* pCmdUI) {
    CDrawingView* pView = dynamic_cast<CDrawingView*>(GetActiveView());
    pCmdUI->Enable(pView && pView->CanUndo());
}

void CMainFrame::OnUpdateEditRedo(CCmdUI* pCmdUI) {
    CDrawingView* pView = dynamic_cast<CDrawingView*>(GetActiveView());
    pCmdUI->Enable(pView && pView->CanRedo());
}

// 修改鼠标左键松开消息，使用 Command 模式执行绘制操作
void CDrawingView::OnLButtonUp(UINT nFlags, CPoint point) {
    CView::OnLButtonUp(nFlags, point);
    if (m_pCurrentShape) {
        m_pCurrentShape->SetPoints(m_startPoint, point);
        // 创建绘制命令并执行
        auto pCommand = std::make_unique<DrawShapeCommand>(this, std::move(m_pCurrentShape));
        ExecuteCommand(std::move(pCommand));
        m_pCurrentShape.reset(); // 释放当前图形指针
    }
    ReleaseCapture();
}
```

关键说明：
1. ICommand 接口定义了 Execute、Undo、Redo 三个核心方法，所有具体命令类均需实现该接口，确保命令的统一管理。
2. DrawShapeCommand 封装了图形绘制的操作，持有绘图视图和图形实例，Undo 方法移除图形，Redo 方法重新添加图形，实现了操作的可逆性。
3. CDrawingView 中通过两个栈（m_undoStack、m_redoStack）管理命令，执行新命令时清空重做栈，符合用户对撤销/重做的常规认知。
4. 主窗口添加了快捷键响应和命令状态更新，当撤销栈/重做栈为空时，对应的菜单选项会置灰，提升用户体验。
5. 命令模式的应用将图形绘制操作与视图逻辑解耦，后续可扩展其他命令（如删除图形、修改图形属性），无需修改核心视图代码。

#### Diff（代码修改与整合）
1. 优化了 DrawShapeCommand 类的资源管理，在构造函数中添加了对 m_pDrawingView 的非空检查，避免空指针访问。
2. 在 CDrawingView 的 ExecuteCommand 方法中添加了命令执行日志记录，将命令描述写入日志文件，方便调试和问题追踪。
3. 调整了 Undo/Redo 栈的实现，使用 std::deque 替代 std::stack，方便后续扩展命令历史记录的查看功能。
4. 补充了快捷键的注册逻辑，在 CMainFrame 的 OnCreate 方法中添加了 ACCELERATOR 表的加载，确保 Ctrl+Z 和 Ctrl+Y 快捷键正常工作。
5. 修复了自由曲线绘制时的撤销/重做逻辑，确保 CFreeCurveShape 的所有临时点都能正确保存和恢复。

### 2025.12.18 16:30 - 17:50
#### Prompt
需要实现多语言支持功能，要求界面支持中、英两种语言切换，包括菜单、工具栏、属性面板的文字显示。请提供 MFC 多语言资源的配置方法、语言切换的核心代码，以及如何整合到现有绘图板项目中，确保切换过程中界面无乱码、功能正常。

#### Response
```cpp
// 1. 多语言资源配置说明
// 步骤1：在项目中添加多语言资源文件（.rc2）
// - 中文资源文件：Resource.zh-CN.rc2
// - 英文资源文件：Resource.en-US.rc2
// 步骤2：在资源文件中定义不同语言的字符串表、菜单、对话框资源
// 示例（中文字符串表，ID为IDS_MENU_DRAW）：
// IDS_MENU_DRAW "绘图(&D)"
// 示例（英文字符串表，ID为IDS_MENU_DRAW）：
// IDS_MENU_DRAW "Draw(&D)"

// 2. 语言管理类（单例模式）
class LanguageManager {
public:
    // 语言类型枚举
    enum LanguageType {
        LT_CHINESE,
        LT_ENGLISH
    };
    
    // 获取单例实例
    static LanguageManager& GetInstance() {
        static LanguageManager instance;
        return instance;
    }
    
    // 禁止拷贝和赋值
    LanguageManager(const LanguageManager&) = delete;
    LanguageManager& operator=(const LanguageManager&) = delete;
    
    // 设置当前语言
    void SetCurrentLanguage(LanguageType type) {
        if (m_currentLang != type) {
            m_currentLang = type;
            // 发送语言切换消息，通知所有窗口更新
            ::SendMessage(HWND_BROADCAST, WM_LANGUAGE_CHANGE, 0, 0);
        }
    }
    
    // 获取当前语言
    LanguageType GetCurrentLanguage() const { return m_currentLang; }
    
    // 根据ID获取对应语言的字符串
    CString GetString(UINT nID) const {
        CString str;
        // 根据当前语言加载对应的资源模块
        HMODULE hModule = GetCurrentLanguageModule();
        if (hModule) {
            str.LoadString(hModule, nID);
        } else {
            str.LoadString(nID); // 加载默认资源
        }
        return str;
    }
    
private:
    LanguageManager() : m_currentLang(LT_CHINESE) {}
    
    // 获取当前语言对应的资源模块句柄
    HMODULE GetCurrentLanguageModule() const {
        switch (m_currentLang) {
            case LT_CHINESE:
                return LoadLibrary(_T("Resource.zh-CN.dll"));
            case LT_ENGLISH:
                return LoadLibrary(_T("Resource.en-US.dll"));
            default:
                return NULL;
        }
    }
    
    LanguageType m_currentLang; // 当前语言
};

// 3. 自定义消息：语言切换
#define WM_LANGUAGE_CHANGE (WM_USER + 1001)

// 4. 主窗口响应语言切换消息
BEGIN_MESSAGE_MAP(CMainFrame, CFrameWndEx)
    // 原有消息映射省略...
    ON_MESSAGE(WM_LANGUAGE_CHANGE, &CMainFrame::OnLanguageChange)
END_MESSAGE_MAP()

LRESULT CMainFrame::OnLanguageChange(WPARAM wParam, LPARAM lParam) {
    // 更新菜单
    UpdateMenu();
    // 更新工具栏
    m_wndToolBar.UpdateDialogControls(this, TRUE);
    // 更新状态栏
    m_wndStatusBar.UpdateWindow();
    // 刷新视图（属性面板等）
    GetActiveView()->InvalidateRect(NULL, FALSE);
    return 0;
}

// 重写主窗口的菜单加载逻辑，使用多语言字符串
void CMainFrame::UpdateMenu() {
    CMenu menu;
    menu.LoadMenu(IDR_MAINFRAME);
    SetMenu(&menu);
    menu.Detach();
    
    // 遍历菜单，更新文字
    CMenu* pMainMenu = GetMenu();
    if (pMainMenu) {
        // 更新“绘图”菜单
        CMenu* pDrawMenu = pMainMenu->GetSubMenu(1); // 假设绘图菜单是第二个菜单项
        if (pDrawMenu) {
            pDrawMenu->ModifyMenu(ID_SHAPE_LINE, MF_BYCOMMAND, ID_SHAPE_LINE, 
                                 LanguageManager::GetInstance().GetString(IDS_SHAPE_LINE));
            pDrawMenu->ModifyMenu(ID_SHAPE_RECT, MF_BYCOMMAND, ID_SHAPE_RECT, 
                                 LanguageManager::GetInstance().GetString(IDS_SHAPE_RECT));
            pDrawMenu->ModifyMenu(ID_SHAPE_ELLIPSE, MF_BYCOMMAND, ID_SHAPE_ELLIPSE, 
                                 LanguageManager::GetInstance().GetString(IDS_SHAPE_ELLIPSE));
            pDrawMenu->ModifyMenu(ID_SHAPE_FREE, MF_BYCOMMAND, ID_SHAPE_FREE, 
                                 LanguageManager::GetInstance().GetString(IDS_SHAPE_FREE));
        }
        
        // 更新“编辑”菜单（撤销/重做）
        CMenu* pEditMenu = pMainMenu->GetSubMenu(2);
        if (pEditMenu) {
            pEditMenu->ModifyMenu(ID_EDIT_UNDO, MF_BYCOMMAND, ID_EDIT_UNDO, 
                                 LanguageManager::GetInstance().GetString(IDS_EDIT_UNDO));
            pEditMenu->ModifyMenu(ID_EDIT_REDO, MF_BYCOMMAND, ID_EDIT_REDO, 
                                 LanguageManager::GetInstance().GetString(IDS_EDIT_REDO));
        }
    }
}

// 5. 属性面板（假设为对话框类 CPropertyDlg）响应语言切换
BEGIN_MESSAGE_MAP(CPropertyDlg, CDialogEx)
    ON_MESSAGE(WM_LANGUAGE_CHANGE, &CPropertyDlg::OnLanguageChange)
END_MESSAGE_MAP()

LRESULT CPropertyDlg::OnLanguageChange(WPARAM wParam, LPARAM lParam) {
    // 更新属性面板控件文字
    SetDlgItemText(IDC_STATIC_COLOR, LanguageManager::GetInstance().GetString(IDS_STATIC_COLOR));
    SetDlgItemText(IDC_STATIC_LINEWIDTH, LanguageManager::GetInstance().GetString(IDS_STATIC_LINEWIDTH));
    SetDlgItemText(IDOK, LanguageManager::GetInstance().GetString(IDS_OK));
    SetDlgItemText(IDCANCEL, LanguageManager::GetInstance().GetString(IDS_CANCEL));
    return 0;
}

// 6. 语言切换菜单响应（在主窗口中添加“语言”菜单）
void CMainFrame::OnLanguageChinese() {
    LanguageManager::GetInstance().SetCurrentLanguage(LanguageManager::LT_CHINESE);
}

void CMainFrame::OnLanguageEnglish() {
    LanguageManager::GetInstance().SetCurrentLanguage(LanguageManager::LT_ENGLISH);
}

BEGIN_MESSAGE_MAP(CMainFrame, CFrameWndEx)
    ON_COMMAND(ID_LANGUAGE_CHINESE, &CMainFrame::OnLanguageChinese)
    ON_COMMAND(ID_LANGUAGE_ENGLISH, &CMainFrame::OnLanguageEnglish)
END_MESSAGE_MAP()
```

关键说明：
1. LanguageManager 采用单例模式，统一管理当前语言和资源加载，确保整个应用的语言一致性。
2. 通过自定义消息 WM_LANGUAGE_CHANGE 实现语言切换时的界面全局更新，包括菜单、工具栏、对话框等所有UI组件。
3. 多语言资源通过独立的 .rc2 文件管理，编译后生成对应的语言资源DLL，避免不同语言资源冲突，便于后续扩展其他语言（如日语、法语）。
4. 字符串加载通过 LanguageManager::GetString 方法统一获取，根据当前语言自动切换资源模块，确保无乱码。
5. 菜单和控件文字的更新逻辑封装在对应窗口类中，符合MFC的消息驱动架构，代码结构清晰。

#### Diff（代码修改与整合）
1. 在项目中添加了多语言资源文件（Resource.zh-CN.rc2 和 Resource.en-US.rc2），定义了所有UI组件的中、英文字符串ID。
2. 优化了 LanguageManager 类的资源模块加载逻辑，添加了 DLL 加载失败的异常处理，确保在缺失语言DLL时能降级使用默认资源。
3. 补充了属性面板的初始化代码，在对话框构造函数中调用 LanguageManager 获取初始语言的字符串，确保程序启动时显示正确的语言。
4. 调整了菜单遍历逻辑，使用递归方式遍历所有子菜单，避免遗漏多级菜单的语言更新。
5. 添加了语言切换的日志记录，将切换时间、切换前后的语言类型写入日志文件，便于问题排查。

