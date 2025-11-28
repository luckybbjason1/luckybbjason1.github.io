import React, { useState, useCallback, useEffect } from 'react';

// --- Global API Configuration Placeholder ---
// The actual key will be managed via state/localStorage by the user input.
const GEMINI_MODEL = 'gemini-2.5-flash-preview-09-2025';
const MAX_RETRIES = 3;

// --- Utility: Exponential Backoff Retry Function ---
const fetchWithRetry = async (url, options) => {
  for (let attempt = 0; attempt < MAX_RETRIES; attempt++) {
    try {
      const response = await fetch(url, options);
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      return await response.json();
    } catch (error) {
      console.error(`Attempt ${attempt + 1} failed: ${error.message}`);
      if (attempt < MAX_RETRIES - 1) {
        const delay = Math.pow(2, attempt) * 1000 + Math.random() * 1000;
        await new Promise(resolve => setTimeout(resolve, delay));
      } else {
        throw new Error('All API attempts failed.');
      }
    }
  }
};

// --- Structured Data Schema for SEO Analysis (Used by existing tool) ---
const SEO_RESPONSE_SCHEMA = {
  type: "OBJECT",
  properties: {
    targetTopic: { type: "STRING", description: "The primary topic analyzed." },
    relatedKeywords: {
      type: "ARRAY",
      description: "3-5 high-relevance, secondary keywords for content targeting, based on current search trends (SERP).",
      items: { type: "STRING" }
    },
    contentStructure: {
      type: "ARRAY",
      description: "Recommended H2 sections and their purpose based on analysis of top 5 ranking pages.",
      items: {
        type: "OBJECT",
        properties: {
          sectionTitle: { type: "STRING", description: "Recommended H2 title in Chinese." },
          coverageGoal: { type: "STRING", description: "Brief explanation in Chinese of what this section must cover to be competitive and satisfy user intent." }
        },
        propertyOrdering: ["sectionTitle", "coverageGoal"]
      }
    }
  },
  propertyOrdering: ["targetTopic", "relatedKeywords", "contentStructure"]
};

// --- Tool Definitions (18 Total, 5 Categories) ---
const toolCategories = [
    { 
        name: '关键词研究', 
        icon: '🔑', 
        color: 'text-purple-600',
        tools: [
            { id: 'content-structure-analyzer', name: '内容结构分析器', isCore: true },
            { id: 'long-tail-generator', name: '长尾关键词生成' },
            { id: 'search-intent-identifier', name: '搜索意图识别' },
            { id: 'keyword-clustering', name: '关键词聚类工具' },
        ]
    },
    { 
        name: '内容优化', 
        icon: '📝', 
        color: 'text-blue-600',
        tools: [
            { id: 'readability-scorer', name: '内容可读性评分' },
            { id: 'semantic-checker', name: '语义相关性检查' },
            { id: 'meta-tag-optimizer', name: '元标签优化' },
            { id: 'title-description-generator', name: '标题/描述生成' },
        ]
    },
    { 
        name: '竞争情报', 
        icon: '📈', 
        color: 'text-green-600',
        tools: [
            { id: 'keyword-gap-analysis', name: '关键词差距分析' },
            { id: 'trending-content-tracker', name: '热门内容追踪' },
            { id: 'backlink-overview', name: '链接概览分析' },
            { id: 'tech-stack-detector', name: '网站技术栈侦测' },
        ]
    },
    { 
        name: '技术SEO', 
        icon: '⚙️', 
        color: 'text-yellow-600',
        tools: [
            { id: 'crawl-budget-advisor', name: '爬行预算优化建议' },
            { id: 'site-speed-analyzer', name: '网站速度分析' },
            { id: 'robots-txt-generator', name: 'Robots.txt 生成器' },
            { id: 'sitemap-validator', name: '网站地图检查' },
        ]
    },
    { 
        name: '策略与报告', 
        icon: '📊', 
        color: 'text-red-600',
        tools: [
            { id: 'quarterly-report-generator', name: '季度SEO报告生成' },
            { id: 'local-seo-audit', name: '本地SEO审计' },
        ]
    },
];

// --- Sub-Component: API Key Configuration ---
const ApiKeyConfig = ({ apiKey, setApiKey, isKeyValid }) => {
    const handleSave = () => {
        localStorage.setItem('geminiApiKey', apiKey.trim());
        alert('API Key 已保存到本地浏览器存储中！');
    };

    return (
        <div className="bg-white p-6 rounded-xl shadow-lg border-l-4 border-yellow-500 mb-8">
            <h2 className="text-xl font-semibold text-gray-800 mb-4 flex items-center">
                <span className="text-yellow-500 mr-2">🔑</span> Gemini API Key 配置
                <span className={`ml-3 text-xs font-bold px-2 py-0.5 rounded-full ${isKeyValid ? 'bg-green-100 text-green-700' : 'bg-red-100 text-red-700'}`}>
                    {isKeyValid ? '已配置' : '未配置/缺失'}
                </span>
            </h2>
            <div className="flex flex-col sm:flex-row gap-3">
                <input
                    type="password"
                    value={apiKey}
                    onChange={(e) => setApiKey(e.target.value)}
                    placeholder="在此粘贴您的 Gemini API Key..."
                    className="flex-grow p-3 border border-gray-300 rounded-lg focus:ring-yellow-500 focus:border-yellow-500 transition duration-150"
                />
                <button
                    onClick={handleSave}
                    disabled={!apiKey.trim()}
                    className="bg-yellow-600 text-white p-3 rounded-lg font-medium hover:bg-yellow-700 transition duration-150 active:scale-95 disabled:bg-yellow-300 min-w-[100px]"
                >
                    保存 Key
                </button>
            </div>
            <p className="mt-2 text-sm text-gray-500">
                Key 将保存在您的浏览器本地存储中，以便于后续使用。
            </p>
        </div>
    );
};


// --- Sub-Component: The 18 Tools Content Renderer ---

// 1. Existing Core Tool Logic (Content Structure Analyzer)
const ContentStructureAnalyzer = ({ apiKey, isLoading, setIsLoading, setError, analysisResult, setAnalysisResult, topic, setTopic }) => {
    
    // Function to perform the SEO analysis
    const performAnalysis = useCallback(async (e) => {
        e.preventDefault();
        if (!apiKey) {
            setError("请先在上方配置您的 Gemini API Key。");
            return;
        }
        if (!topic.trim()) {
            setError("请输入您想要分析的内容主题或核心关键词。");
            return;
        }
        
        setIsLoading(true);
        setError(null);
        setAnalysisResult(null);

        const GEMINI_API_URL = `https://generativelanguage.googleapis.com/v1beta/models/${GEMINI_MODEL}:generateContent?key=${apiKey}`;

        const systemPrompt = `你是一名顶级的SEO分析师和内容策略师。你的任务是根据当前互联网上的搜索结果（SERP），为用户的主题生成一份内容大纲和关键词洞察。
        1. 确定3-5个与主题高度相关、具有长尾潜力的二级关键词。
        2. 分析竞争对手，生成一套推荐的H2内容结构（至少5个H2部分），并简要说明每个部分的覆盖目标。
        3. 你的回复必须严格遵循提供的 JSON Schema，并用中文输出。`;
        
        // English query is crucial for Google Search grounding reliability
        const userQuery = `请分析以下主题，并提供SEO内容结构大纲和关键词洞察：${topic}`;

        const payload = {
          contents: [{ parts: [{ text: userQuery }] }],
          tools: [{ "google_search": {} }],
          systemInstruction: {
            parts: [{ text: systemPrompt }]
          },
          generationConfig: {
            responseMimeType: "application/json",
            responseSchema: SEO_RESPONSE_SCHEMA
          }
        };

        try {
          const result = await fetchWithRetry(GEMINI_API_URL, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(payload)
          });
          
          const jsonText = result.candidates?.[0]?.content?.parts?.[0]?.text;
          if (!jsonText) {
             throw new Error("API返回结构为空或解析失败。");
          }
          
          const parsedJson = JSON.parse(jsonText);
          setAnalysisResult(parsedJson);

        } catch (e) {
          console.error("SEO API Analysis Failed:", e);
          setError(`分析失败：无法获取结构化数据。请检查您的 API Key 或网络连接。`);
        } finally {
          setIsLoading(false);
        }
    }, [topic, apiKey, setIsLoading, setError, setAnalysisResult]);

    // UI Component for Data Visualization (Structured Insights)
    const AnalysisDisplay = ({ data }) => {
        if (!data) return null;

        return (
          <div className="mt-8 space-y-8">
            {/* Keywords Card */}
            <div className="bg-white p-6 rounded-xl shadow-lg border-l-4 border-purple-500 transition hover:shadow-xl">
              <h3 className="text-xl font-bold text-gray-800 mb-3 flex items-center">
                <svg xmlns="http://www.w3.org/2000/svg" className="h-6 w-6 mr-2 text-purple-600" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth={2}>
                  <path strokeLinecap="round" strokeLinejoin="round" d="M13 10V3L4 14h7v7l9-11h-7z" />
                </svg>
                🚀 核心主题洞察: <span className="ml-2 text-purple-600 italic">{data.targetTopic}</span>
              </h3>
              <p className="text-sm text-gray-500 mb-4 border-b pb-3">基于当前搜索意图和竞争环境的分析。</p>
              
              <h4 className="font-semibold text-lg text-gray-700 mb-3">长尾关键词推荐 (LSI)</h4>
              <div className="flex flex-wrap gap-2">
                {data.relatedKeywords?.map((keyword, index) => (
                  <span key={index} className="bg-purple-100 text-purple-700 text-sm font-medium px-4 py-1 rounded-full hover:bg-purple-200 transition cursor-help" title="用于文章中的自然融入">
                    {keyword}
                  </span>
                ))}
              </div>
            </div>

            {/* Content Structure Card */}
            <div className="bg-white p-6 rounded-xl shadow-lg border-l-4 border-blue-500 transition hover:shadow-xl">
              <h3 className="text-xl font-bold text-gray-800 mb-4 flex items-center">
                <svg xmlns="http://www.w3.org/2000/svg" className="h-6 w-6 mr-2 text-blue-600" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth={2}>
                  <path strokeLinecap="round" strokeLinejoin="round" d="M4 6h16M4 10h16M4 14h16M4 18h16" />
                </svg>
                📝 推荐内容大纲 (H2 结构)
              </h3>
              <p className="text-sm text-gray-500 mb-4 border-b pb-3">确保您的内容结构完整且具竞争力。</p>
              
              <div className="space-y-4">
                {data.contentStructure?.map((section, index) => (
                  <div key={index} className="border border-gray-100 p-4 rounded-lg bg-gray-50 hover:bg-white transition duration-200">
                    <p className="text-lg font-semibold text-blue-700 mb-1">
                      {index + 1}. {section.sectionTitle}
                    </p>
                    <p className="text-sm text-gray-600">
                      <span className="font-medium text-gray-500">覆盖目标:</span> {section.coverageGoal}
                    </p>
                  </div>
                ))}
              </div>
            </div>
            
            {/* Disclaimer */}
            <div className="text-center text-xs text-gray-400 pt-4 border-t">
              <p>* 所有数据由AI实时分析当前谷歌搜索结果得出，仅供参考。</p>
            </div>
          </div>
        );
    };

    return (
        <>
            <form onSubmit={performAnalysis} className="flex flex-col sm:flex-row gap-3">
                <input
                  type="text"
                  value={topic}
                  onChange={(e) => setTopic(e.target.value)}
                  placeholder="输入您的核心主题或关键词，例如：'如何选择最好的云计算平台'"
                  className="flex-grow p-3 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500 transition duration-150"
                  required
                  disabled={isLoading}
                />
                <button
                  type="submit"
                  disabled={isLoading}
                  className="bg-blue-600 text-white p-3 rounded-lg font-medium hover:bg-blue-700 transition duration-150 active:scale-95 disabled:bg-blue-300 flex items-center justify-center min-w-[100px]"
                >
                  {isLoading ? (
                    <svg className="animate-spin h-5 w-5 mr-3 text-white" viewBox="0 0 24 24">
                      <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
                      <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                    </svg>
                  ) : (
                    '开始分析'
                  )}
                </button>
            </form>
            <AnalysisDisplay data={analysisResult} />
        </>
    );
};


// 2. Generic Tool Skeleton for New Tools
const GenericToolSkeleton = ({ title, description, placeholder, apiKey, isLoading, setIsLoading, setError }) => {
    const [inputValue, setInputValue] = useState('');
    const [analysisMessage, setAnalysisMessage] = useState(null);

    const handleAnalysis = async (e) => {
        e.preventDefault();
        if (!apiKey) {
            setError("请先在上方配置您的 Gemini API Key。");
            return;
        }
        if (!inputValue.trim()) {
            setError("请输入所需信息。");
            return;
        }

        setIsLoading(true);
        setError(null);
        setAnalysisMessage(null);
        
        // --- AI Call Skeleton ---
        const systemPrompt = `你是一个专业的SEO工具，擅长进行 ${title} 分析。请根据用户的输入: "${inputValue}"，并结合实时搜索数据，提供一份关于 ${title} 的简洁分析报告。`;
        const userQuery = `请针对主题：${inputValue}，执行 ${title} 分析。`;
        const GEMINI_API_URL = `https://generativelanguage.googleapis.com/v1beta/models/${GEMINI_MODEL}:generateContent?key=${apiKey}`;

        const payload = {
            contents: [{ parts: [{ text: userQuery }] }],
            tools: [{ "google_search": {} }],
            systemInstruction: { parts: [{ text: systemPrompt }] },
        };

        try {
            const result = await fetchWithRetry(GEMINI_API_URL, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(payload)
            });

            const text = result.candidates?.[0]?.content?.parts?.[0]?.text || "AI未能生成有效报告。";
            setAnalysisMessage({
                text: text,
                timestamp: new Date().toLocaleTimeString('zh-CN'),
            });
            
        } catch (e) {
            console.error(`${title} API Failed:`, e);
            setError(`[${title}] 分析失败：无法连接到AI服务。`);
        } finally {
            setIsLoading(false);
        }
    };

    return (
        <>
            <p className="text-gray-500 mb-4">{description}</p>
            <form onSubmit={handleAnalysis} className="flex flex-col sm:flex-row gap-3 mb-6">
                <input
                    type="text"
                    value={inputValue}
                    onChange={(e) => setInputValue(e.target.value)}
                    placeholder={placeholder}
                    className="flex-grow p-3 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500 transition duration-150"
                    required
                    disabled={isLoading}
                />
                <button
                    type="submit"
                    disabled={isLoading}
                    className="bg-gray-600 text-white p-3 rounded-lg font-medium hover:bg-gray-700 transition duration-150 active:scale-95 disabled:bg-gray-300 flex items-center justify-center min-w-[100px]"
                >
                    {isLoading ? (
                        <svg className="animate-spin h-5 w-5 mr-3 text-white" viewBox="0 0 24 24">...</svg>
                    ) : (
                        `执行 ${title}`
                    )}
                </button>
            </form>

            {analysisMessage && (
                <div className="bg-green-50 p-6 rounded-xl border border-green-200">
                    <h4 className="text-lg font-bold text-green-800 mb-2">✅ {title} 报告</h4>
                    <p className="whitespace-pre-wrap text-gray-700">{analysisMessage.text}</p>
                    <p className="text-xs text-gray-500 mt-3 border-t pt-2">报告生成时间: {analysisMessage.timestamp}</p>
                </div>
            )}

            {!isLoading && !analysisMessage && (
                <div className="bg-white p-8 rounded-xl shadow-inner border border-gray-100 text-center text-gray-400 min-h-[150px] flex items-center justify-center">
                    <p>请输入信息并点击执行按钮，以生成 {title} 报告。</p>
                </div>
            )}
        </>
    );
};


// Main Tool Selector Component
const ToolContent = ({ activeTool, apiKey, isLoading, setIsLoading, setError, analysisResult, setAnalysisResult, topic, setTopic }) => {
    const allTools = toolCategories.flatMap(cat => cat.tools);
    const tool = allTools.find(t => t.id === activeTool);

    if (!tool) {
        return <div className="p-8 text-center text-gray-500">请从左侧选择一个 SEO 工具开始工作。</div>;
    }

    const toolProps = { apiKey, isLoading, setIsLoading, setError };

    switch (tool.id) {
        case 'content-structure-analyzer':
            return <ContentStructureAnalyzer {...toolProps} analysisResult={analysisResult} setAnalysisResult={setAnalysisResult} topic={topic} setTopic={setTopic} />;
        
        // --- Keyword Research
        case 'long-tail-generator':
            return <GenericToolSkeleton title={tool.name} description="根据您的核心主题，生成一系列有潜力且低竞争的长尾关键词。" placeholder="输入核心主题，例如：'自托管云存储方案'" {...toolProps} />;
        case 'search-intent-identifier':
            return <GenericToolSkeleton title={tool.name} description="分析关键词背后的用户意图（信息型、交易型、导航型、商业研究型）。" placeholder="输入关键词，例如：'最佳VPN推荐'" {...toolProps} />;
        case 'keyword-clustering':
            return <GenericToolSkeleton title={tool.name} description="分析一系列关键词列表，并将其聚类为逻辑组，以指导内容集群策略。" placeholder="输入关键词列表，以逗号分隔" {...toolProps} />;

        // --- Content Optimization
        case 'readability-scorer':
            return <GenericToolSkeleton title={tool.name} description="评估文本的可读性分数（如Flesch-Kincaid），并提供改进建议。" placeholder="粘贴您要评估的中文文本内容" {...toolProps} />;
        case 'semantic-checker':
            return <GenericToolSkeleton title={tool.name} description="检查内容中是否覆盖了与目标主题高度相关的 LSI（潜在语义索引）关键词。" placeholder="输入目标关键词和文章内容链接或摘要" {...toolProps} />;
        case 'meta-tag-optimizer':
            return <GenericToolSkeleton title={tool.name} description="根据目标关键词和内容摘要，生成具有高点击率潜力的优化元标题和描述。" placeholder="输入目标关键词和内容简述" {...toolProps} />;
        case 'title-description-generator':
            return <GenericToolSkeleton title={tool.name} description="为文章或产品页批量生成多个吸引人的标题和描述变体。" placeholder="输入核心卖点或主题" {...toolProps} />;

        // --- Competitor Intelligence
        case 'keyword-gap-analysis':
            return <GenericToolSkeleton title={tool.name} description="对比您和竞争对手的网站，找出您当前未排名但竞争对手有排名的关键词。" placeholder="输入您的域名和最多3个竞争对手域名" {...toolProps} />;
        case 'trending-content-tracker':
            return <GenericToolSkeleton title={tool.name} description="找出您所在行业或利基市场当前最热门和最快增长的内容趋势。" placeholder="输入您的行业或利基市场" {...toolProps} />;
        case 'backlink-overview':
            return <GenericToolSkeleton title={tool.name} description="分析指定域名的外部链接概况，评估链接质量和数量。" placeholder="输入要分析的域名或URL" {...toolProps} />;
        case 'tech-stack-detector':
            return <GenericToolSkeleton title={tool.name} description="快速识别竞争对手网站使用的主要技术栈（CMS, 框架, 缓存等）。" placeholder="输入竞争对手的域名" {...toolProps} />;

        // --- Technical SEO
        case 'crawl-budget-advisor':
            return <GenericToolSkeleton title={tool.name} description="根据您的网站大小和爬行统计，提供优化爬行预算的策略建议。" placeholder="输入您的域名和页面数量" {...toolProps} />;
        case 'site-speed-analyzer':
            return <GenericToolSkeleton title={tool.name} description="（此工具通常需要第三方API集成，此处为占位）分析网页速度核心指标 (Core Web Vitals) 并提供优化建议。" placeholder="输入要分析的网页URL" {...toolProps} />;
        case 'robots-txt-generator':
            return <GenericToolSkeleton title={tool.name} description="根据您的需求生成一个优化的 robots.txt 文件，以指导搜索引擎爬虫。" placeholder="输入需要禁止爬取的路径（如 /admin/）" {...toolProps} />;
        case 'sitemap-validator':
            return <GenericToolSkeleton title={tool.name} description="检查您的 XML 网站地图格式是否正确，并提供改进建议。" placeholder="输入您的网站地图URL" {...toolProps} />;
        
        // --- Strategy & Reporting
        case 'quarterly-report-generator':
            return <GenericToolSkeleton title={tool.name} description="根据过去三个月的关键SEO数据（假定输入），生成一份简洁专业的季度报告和下一步战略建议。" placeholder="输入季度关键词、流量和排名摘要数据" {...toolProps} />;
        case 'local-seo-audit':
            return <GenericToolSkeleton title={tool.name} description="为特定的本地业务（如餐馆、律师事务所）执行完整的本地SEO审计和改进清单。" placeholder="输入公司名称和城市/地区" {...toolProps} />;
        
        default:
            return <div className="p-8 text-center text-gray-500">未找到选定的工具。</div>;
    }
};


// --- Main Application Component ---
const App = () => {
  const [activeTool, setActiveTool] = useState('content-structure-analyzer');
  const [topic, setTopic] = useState(''); // Input for core tool
  const [analysisResult, setAnalysisResult] = useState(null); // Result for core tool
  
  const [apiKey, setApiKey] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState(null);

  // Load API Key from localStorage on mount
  useEffect(() => {
    const storedKey = localStorage.getItem('geminiApiKey');
    if (storedKey) {
        setApiKey(storedKey);
    }
  }, []);

  const isKeyValid = apiKey.trim().length > 10;
  
  // Reset states when switching tools
  useEffect(() => {
    setError(null);
    setIsLoading(false);
    setAnalysisResult(null);
  }, [activeTool]);


  return (
    <div className="min-h-screen bg-gray-50 p-0 font-sans antialiased flex">

      {/* --- Sidebar (Navigation) --- */}
      <div className="w-1/4 min-w-[280px] bg-white border-r border-gray-100 p-6 shadow-xl sticky top-0 h-screen overflow-y-auto hidden md:block">
        <div className="mb-8">
            <h1 className="text-3xl font-extrabold text-blue-700 tracking-tight">
                Insight Engine
            </h1>
            <p className="text-sm text-gray-500 mt-1">18 项结构化洞察工具</p>
        </div>

        <nav className="space-y-4">
            {toolCategories.map((category) => (
                <div key={category.name} className="space-y-2">
                    <h3 className={`text-sm font-bold uppercase tracking-wider ${category.color} flex items-center`}>
                        {category.icon}
                        <span className="ml-2">{category.name}</span>
                    </h3>
                    <ul className="space-y-1">
                        {category.tools.map((tool) => (
                            <li key={tool.id}>
                                <button
                                    onClick={() => setActiveTool(tool.id)}
                                    className={`w-full text-left p-3 rounded-lg flex items-center transition duration-150 ${activeTool === tool.id 
                                        ? 'bg-blue-500 text-white font-semibold shadow-md' 
                                        : 'text-gray-700 hover:bg-gray-100'
                                    }`}
                                >
                                    <span className="text-sm font-medium">{tool.name}</span>
                                    {tool.isCore && <span className="ml-auto text-xs bg-white text-blue-500 px-2 py-0.5 rounded-full font-bold">核心</span>}
                                </button>
                            </li>
                        ))}
                    </ul>
                </div>
            ))}
        </nav>
      </div>

      {/* --- Main Content Area --- */}
      <div className="flex-grow p-4 sm:p-8 overflow-y-auto">
        <div className="max-w-4xl mx-auto">
            
            {/* API Key Configuration - Persists via localStorage */}
            <ApiKeyConfig apiKey={apiKey} setApiKey={setApiKey} isKeyValid={isKeyValid} />

            {/* Current Tool Title */}
            <div className="mb-8">
                <h2 className="text-3xl font-bold text-gray-800 tracking-tight">
                    {toolCategories.flatMap(cat => cat.tools).find(t => t.id === activeTool)?.name || 'SEO 工具'}
                </h2>
                <p className="text-lg text-gray-500 mt-1">
                    {toolCategories.find(cat => cat.tools.some(t => t.id === activeTool))?.name} 
                    {" > "}
                    {toolCategories.flatMap(cat => cat.tools).find(t => t.id === activeTool)?.name}
                </p>
            </div>

            {/* Error Message Display */}
            {error && (
              <div role="alert" className="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded-xl relative mb-8">
                <strong className="font-bold">分析错误!</strong>
                <span className="block sm:inline ml-2">{error}</span>
              </div>
            )}
            
            {/* Tool Content Container */}
            <div className="bg-white p-6 rounded-xl shadow-2xl border border-gray-100">
                <ToolContent 
                    activeTool={activeTool}
                    apiKey={isKeyValid ? apiKey : null} // Only pass key if valid/present
                    isLoading={isLoading}
                    setIsLoading={setIsLoading}
                    setError={setError}
                    analysisResult={analysisResult}
                    setAnalysisResult={setAnalysisResult}
                    topic={topic}
                    setTopic={setTopic}
                />
            </div>

        </div>
      </div>
    </div>
  );
};

export default App;     根据这个网站做一个 后台  我需要 前台有方广告的位置   广告位置 大概 5-10个   不用的时候看不出来    放在合理的位置    前台的布局根 UI  需要优化   SEO 注重   工具的各种功能 API的联动性 要准确   

