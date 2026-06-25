<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>머니바이브 (MoneyVibe) - 대학생 맞춤형 스마트 지출관리 가계부</title>
    <!-- Tailwind CSS CDN -->
    <script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>
    <!-- React & ReactDOM CDN -->
    <script src="https://unpkg.com/react@18/umd/react.development.js" crossorigin></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js" crossorigin></script>
    <!-- Babel CDN for JSX -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <!-- Font Awesome for Icons -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    <!-- Google Fonts -->
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&family=Noto+Sans+KR:wght@300;400;500;700&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Inter', 'Noto Sans KR', sans-serif;
            background-color: #f8fafc;
        }
        /* Custom scrollbar for better UI */
        ::-webkit-scrollbar {
            width: 6px;
            height: 6px;
        }
        ::-webkit-scrollbar-track {
            background: #f1f5f9;
        }
        ::-webkit-scrollbar-thumb {
            background: #cbd5e1;
            border-radius: 4px;
        }
        ::-webkit-scrollbar-thumb:hover {
            background: #94a3b8;
        }
    </style>
</head>
<body class="text-slate-800 antialiased min-h-screen">

    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect, useMemo } = React;

        // 초기 더미 데이터 설정 (대학생 맞춤형 소비 패턴 반영)
        const INITIAL_EXPENSES = [
            { id: 1, amount: 8500, date: '2026-06-15', memo: '학생회관 학식', category: '식비(학식·배달)', paymentMethod: '체크카드' },
            { id: 2, amount: 5500, date: '2026-06-14', memo: '정문 투썸 카공', category: '카페·카공', paymentMethod: '간편결제(카카오·네이버페이)' },
            { id: 3, amount: 28000, date: '2026-06-12', memo: '동아리 종강 술자리', category: '술자리·유흥', paymentMethod: '체크카드' },
            { id: 4, amount: 19500, date: '2026-06-10', memo: '전공교재 중고구매', category: '학업·교재', paymentMethod: '현금' },
            { id: 5, amount: 1400, date: '2026-06-09', memo: '통학 지하철 요금', category: '교통비', paymentMethod: '신용카드' },
        ];

        const CATEGORIES = [
            { name: '식비(학식·배달)', color: '#f59e0b', bg: 'bg-amber-50', text: 'text-amber-700', icon: 'fa-utensils' },
            { name: '카페·카공', color: '#ea580c', bg: 'bg-orange-50', text: 'text-orange-700', icon: 'fa-coffee' },
            { name: '술자리·유흥', color: '#a855f7', bg: 'bg-purple-50', text: 'text-purple-700', icon: 'fa-beer-mushrooms' },
            { name: '학업·교재', color: '#6366f1', bg: 'bg-indigo-50', text: 'text-indigo-700', icon: 'fa-book' },
            { name: '교통비', color: '#3b82f6', bg: 'bg-blue-50', text: 'text-blue-700', icon: 'fa-bus' },
            { name: '쇼핑·문화', color: '#ec4899', bg: 'bg-pink-50', text: 'text-pink-700', icon: 'fa-shirt' }
        ];

        function MoneyVibeApp() {
            // --- State 관리 ---
            const [budget, setBudget] = useState(500000); // 이번 달 총 예산 설정 (기본값 50만원)
            const [isEditingBudget, setIsEditingBudget] = useState(false);
            const [budgetInput, setBudgetInput] = useState(500000);

            const [expenses, setExpenses] = useState(INITIAL_EXPENSES);
            const [sortBy, setSortBy] = useState('latest'); // 'latest' | 'highest'

            // 입력 폼 필드 상태
            const [amount, setAmount] = useState('');
            const [date, setDate] = useState(new Date().toISOString().split('T')[0]); // 오늘 날짜 기본값
            const [memo, setMemo] = useState('');
            const [category, setCategory] = useState('식비(학식·배달)');
            const [paymentMethod, setPaymentMethod] = useState('체크카드');
            
            // 경고 및 알림 메시지 상태
            const [alertMessage, setAlertMessage] = useState(null);

            // --- 연산 로직 (Dashboard 계산) ---
            const totalSpent = useMemo(() => {
                return expenses.reduce((sum, item) => sum + item.amount, 0);
            }, [expenses]);

            const remainingBudget = budget - totalSpent;
            const remainingPercentage = budget > 0 ? (remainingBudget / budget) * 100 : 0;
            
            // Rule 기반 조건부 렌더링 트리거: 남은 용돈이 설정 예산의 20% 미만인지 확인
            const isEmergency = remainingPercentage < 20;

            // --- 정렬 필터 적용 ---
            const sortedExpenses = useMemo(() => {
                const copied = [...expenses];
                if (sortBy === 'latest') {
                    return copied.sort((a, b) => new Date(b.date) - new Date(a.date) || b.id - a.id);
                } else if (sortBy === 'highest') {
                    return copied.sort((a, b) => b.amount - a.amount);
                }
                return copied;
            }, [expenses, sortBy]);

            // --- 카테고리별 통계 집계 오케스트레이션 ---
            const categoryStats = useMemo(() => {
                // 각 카테고리별 합산금액 초기화
                const statsMap = {};
                CATEGORIES.forEach(cat => {
                    statsMap[cat.name] = { amount: 0, percentage: 0, meta: cat };
                });

                // 전체 지출 배열을 순회하며 카테고리별 속성 금액 합산
                expenses.forEach(item => {
                    if (statsMap[item.category]) {
                        statsMap[item.category].amount += item.amount;
                    }
                });

                // 점유 비율(%) 계산 수식 적용
                CATEGORIES.forEach(cat => {
                    if (totalSpent > 0) {
                        statsMap[cat.name].percentage = Math.round((statsMap[cat.name].amount / totalSpent) * 100);
                    }
                });

                return Object.values(statsMap).sort((a, b) => b.amount - a.amount);
            }, [expenses, totalSpent]);

            // --- 핸들러 함수들 ---
            const handleAddExpense = (e) => {
                e.preventDefault();

                // 1-1. 기본 데이터 검증 및 인터랙션 처리
                if (!amount || parseInt(amount) <= 0) {
                    showAlert("금액을 정확히 입력해 주세요.");
                    return;
                }
                if (memo.trim().length === 0) {
                    showAlert("사용처나 품목 메모를 입력해 주세요.");
                    return;
                }
                if (memo.length > 20) {
                    showAlert("메모는 최대 20자까지만 입력 가능합니다.");
                    return;
                }

                // 새로운 지출 객체 정렬 생성
                const newExpense = {
                    id: Date.now(),
                    amount: parseInt(amount),
                    date: date,
                    memo: memo.trim(),
                    category: category,
                    paymentMethod: paymentMethod
                };

                // 가계부 내역 상태 배열에 추가(push 대체형 state 전개 연산자)
                setExpenses([newExpense, ...expenses]);

                // 입력 폼 초기화 (날짜는 유지, 금액과 메모 비우기)
                setAmount('');
                setMemo('');
                showAlert("✨ 지출 내역이 성공적으로 저장되었습니다!");
            };

            const handleDeleteExpense = (id) => {
                // 고유 ID를 가진 데이터를 배열에서 제거(filter)
                setExpenses(expenses.filter(item => item.id !== id));
            };

            const handleUpdateBudget = (e) => {
                e.preventDefault();
                const val = parseInt(budgetInput);
                if (val >= 0) {
                    setBudget(val);
                    setIsEditingBudget(false);
                    showAlert("🎯 이번 달 목표 예산이 변경되었습니다.");
                }
            };

            const showAlert = (msg) => {
                setAlertMessage(msg);
                setTimeout(() => setAlertMessage(null), 3000);
            };

            // SVG 파이 차트용 각도 계산 함수들
            let cumulativePercent = 0;
            const getCoordinatesForPercent = (percent) => {
                const x = Math.cos(2 * Math.PI * percent);
                const y = Math.sin(2 * Math.PI * percent);
                return [x, y];
            };

            return (
                <div class="max-w-7xl mx-auto px-4 py-6 md:py-10">
                    {/* 상단 헤더 섹션 (과제 제출 정보 반영) */}
                    <header class="mb-8 flex flex-col md:flex-row md:items-center md:justify-between border-b border-slate-200 pb-5 bg-white p-6 rounded-2xl shadow-xs border border-slate-100">
                        <div>
                            <div class="flex items-center gap-2 mb-1">
                                <span class="bg-indigo-600 text-white text-xs font-semibold px-2.5 py-1 rounded-sm uppercase tracking-wide">SW 프로그래밍 기초 과제</span>
                                <span class="text-slate-500 text-xs">한문학과 이창형 (2026130704)</span>
                            </div>
                            <h1 class="text-2xl md:text-3xl font-bold tracking-tight text-slate-900 flex items-center gap-2">
                                <i class="fa-solid fa-wallet text-indigo-600"></i> 머니바이브 <span class="text-indigo-600 font-medium text-lg md:text-xl">MoneyVibe</span>
                            </h1>
                            <p class="text-slate-500 text-sm mt-1">대학생을 위한 초간편 직관적 스마트 지출 관리 대시보드 시스템</p>
                        </div>
                        <div class="mt-4 md:mt-0 flex gap-2 text-xs text-slate-400 bg-slate-50 p-3 rounded-lg border border-slate-100 self-start md:self-auto">
                            <div><span class="font-medium text-slate-600">Framework:</span> VibeCoding Standard</div>
                            <span class="text-slate-300">|</span>
                            <div><span class="font-medium text-slate-600">Stack:</span> React + Tailwind CSS</div>
                        </div>
                    </header>

                    {/* 글로벌 알림 팝업 토스트 */}
                    {alertMessage && (
                        <div class="fixed top-5 left-1/2 transform -translate-x-1/2 z-50 bg-slate-900 text-white px-5 py-3 rounded-xl shadow-xl flex items-center gap-3 text-sm animate-bounce">
                            <i class="fa-solid fa-circle-info text-indigo-400"></i>
                            <span>{alertMessage}</span>
                        </div>
                    )}

                    {/* 메인 대시보드 및 서비스 환경 그리드 */}
                    <main class="grid grid-cols-1 lg:grid-cols-12 gap-6 items-start">
                        
                        {/* LEFT COLUMN: 입력 양식 및 예산 컨트롤 (4개 열 공간 차지) */}
                        <section class="lg:col-span-4 space-y-6">
                            
                            {/* [예산 설정 컴포넌트] */}
                            <div class="bg-white p-6 rounded-2xl shadow-xs border border-slate-100">
                                <div class="flex justify-between items-center mb-4">
                                    <h3 class="font-bold text-slate-900 flex items-center gap-2">
                                        <i class="fa-solid fa-sliders text-slate-400"></i> 예산 한도 설정
                                    </h3>
                                    {!isEditingBudget && (
                                        <button onClick={() => { setBudgetInput(budget); setIsEditingBudget(true); }} class="text-xs text-indigo-600 hover:underline flex items-center gap-1">
                                            <i class="fa-solid fa-pen"></i> 수정하기
                                        </button>
                                    )}
                                </div>
                                {isEditingBudget ? (
                                    <form onSubmit={handleUpdateBudget} class="flex gap-2">
                                        <input 
                                            type="number" 
                                            value={budgetInput}
                                            onChange={(e) => setBudgetInput(e.target.value)}
                                            class="w-full px-3 py-1.5 border border-slate-200 rounded-lg text-sm focus:outline-hidden focus:ring-2 focus:ring-indigo-500"
                                            placeholder="금액 입력 (원)"
                                            min="0"
                                            required
                                        />
                                        <button type="submit" class="bg-indigo-600 text-white text-xs px-3 py-1.5 rounded-lg font-medium hover:bg-indigo-700 whitespace-nowrap">적용</button>
                                        <button type="button" onClick={() => setIsEditingBudget(false)} class="bg-slate-100 text-slate-600 text-xs px-3 py-1.5 rounded-lg font-medium hover:bg-slate-200">취소</button>
                                    </form>
                                ) : (
                                    <div class="bg-slate-50 p-3 rounded-xl flex justify-between items-center">
                                        <span class="text-xs text-slate-500">지정된 월간 목표 예산:</span>
                                        <span class="font-bold text-slate-800">{budget.toLocaleString()}원</span>
                                    </div>
                                )}
                            </div>

                            {/* [기능 1] 지출 내역 입력 양식 서브 섹션 */}
                            <div class="bg-white p-6 rounded-2xl shadow-xs border border-slate-100">
                                <h3 class="font-bold text-slate-900 mb-4 flex items-center gap-2">
                                    <i class="fa-solid fa-circle-plus text-indigo-600"></i> 신규 지출 내역 입력
                                </h3>
                                <form onSubmit={handleAddExpense} class="space-y-4">
                                    
                                    {/* 금액 입력창 (1-1) */}
                                    <div>
                                        <label class="block text-xs font-semibold text-slate-600 mb-1">지출 금액 <span class="text-rose-500">*</span></label>
                                        <div class="relative">
                                            <input 
                                                type="number" 
                                                value={amount}
                                                onChange={(e) => setAmount(e.target.value)}
                                                placeholder="금액을 입력해 주세요" 
                                                class="w-full pl-3 pr-10 py-2 border border-slate-200 rounded-xl text-sm focus:outline-hidden focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 font-medium"
                                            />
                                            <span class="absolute right-3 top-2.5 text-slate-400 text-xs">원</span>
                                        </div>
                                    </div>

                                    {/* 날짜 피커 (1-1) */}
                                    <div>
                                        <label class="block text-xs font-semibold text-slate-600 mb-1">결제 일자 <span class="text-rose-500">*</span></label>
                                        <input 
                                            type="date" 
                                            value={date}
                                            onChange={(e) => setDate(e.target.value)}
                                            class="w-full px-3 py-2 border border-slate-200 rounded-xl text-sm focus:outline-hidden focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 text-slate-700"
                                        />
                                    </div>

                                    {/* 카테고리 드롭다운 분류 (1-2) */}
                                    <div>
                                        <label class="block text-xs font-semibold text-slate-600 mb-1">소비 카테고리</label>
                                        <select 
                                            value={category}
                                            onChange={(e) => setCategory(e.target.value)}
                                            class="w-full px-3 py-2 border border-slate-200 rounded-xl text-sm bg-white focus:outline-hidden focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 text-slate-700"
                                        >
                                            {CATEGORIES.map(cat => (
                                                <option key={cat.name} value={cat.name}>{cat.name}</option>
                                            ))}
                                        </select>
                                    </div>

                                    {/* 결제 수단 라디오 버튼 분류 (1-2) */}
                                    <div>
                                        <label class="block text-xs font-semibold text-slate-600 mb-1">결제 수단</label>
                                        <div class="grid grid-cols-2 gap-2">
                                            {['체크카드', '신용카드', '현금', '간편결제'].map((method) => {
                                                const fullMethodName = method === '간편결제' ? '간편결제(카카오·네이버페이)' : method;
                                                return (
                                                    <label key={method} class={`flex items-center gap-2 p-2 border rounded-xl text-xs cursor-pointer transition-colors ${paymentMethod === fullMethodName ? 'border-indigo-600 bg-indigo-50/50 font-semibold text-indigo-700' : 'border-slate-200 hover:bg-slate-50 text-slate-600'}`}>
                                                        <input 
                                                            type="radio" 
                                                            name="paymentMethod" 
                                                            value={fullMethodName} 
                                                            checked={paymentMethod === fullMethodName}
                                                            onChange={() => setPaymentMethod(fullMethodName)}
                                                            class="text-indigo-600 focus:ring-indigo-500 h-3.5 w-3.5"
                                                        />
                                                        <span>{method}</span>
                                                    </label>
                                                );
                                            })}
                                        </div>
                                    </div>

                                    {/* 메모창 20자 글자 제한 (1-1) */}
                                    <div>
                                        <div class="flex justify-between items-center mb-1">
                                            <label class="block text-xs font-semibold text-slate-600">내역 메모 <span class="text-rose-500">*</span></label>
                                            <span class={`text-[10px] ${memo.length > 20 ? 'text-rose-500 font-bold' : 'text-slate-400'}`}>{memo.length}/20자</span>
                                        </div>
                                        <input 
                                            type="text" 
                                            value={memo}
                                            onChange={(e) => setMemo(e.target.value.slice(0, 20))}
                                            placeholder="사용처 또는 품목 입력 (20자 이내)" 
                                            maxLength="20"
                                            class="w-full px-3 py-2 border border-slate-200 rounded-xl text-sm focus:outline-hidden focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500"
                                        />
                                    </div>

                                    {/* 저장하기 트리거 버튼 (1-3) */}
                                    <button 
                                        type="submit"
                                        class="w-full bg-slate-900 hover:bg-indigo-600 text-white font-medium text-sm py-2.5 rounded-xl transition-all duration-200 flex items-center justify-center gap-2 shadow-xs cursor-pointer"
                                    >
                                        <i class="fa-solid fa-floppy-disk"></i> 지출 내역 저장하기
                                    </button>
                                </form>
                            </div>
                        </section>

                        {/* RIGHT COLUMN: 대시보드 스태츠 및 그래프 통계 리스트 (8개 열 공간 차지) */}
                        <section class="lg:col-span-8 space-y-6">
                            
                            {/* [기능 2-1, 2-2] 실시간 대시보드 카드 연산 및 조건부 렌더링 */}
                            <div class={`p-6 rounded-2xl border transition-all duration-300 shadow-xs ${
                                isEmergency 
                                ? 'bg-rose-50 border-rose-200 text-rose-900' 
                                : 'bg-emerald-50 border-emerald-200 text-emerald-900'
                            }`}>
                                <div class="flex flex-col md:flex-row justify-between items-start md:items-center gap-4">
                                    <div>
                                        <div class="flex items-center gap-2">
                                            <span class={`px-2 py-0.5 rounded-sm text-[10px] font-bold uppercase tracking-wider ${isEmergency ? 'bg-rose-600 text-white' : 'bg-emerald-600 text-white'}`}>
                                                {isEmergency ? '⚠️ 소비 경고' : '✓ 안정 자산'}
                                            </span>
                                            <h2 class="text-sm font-bold tracking-wide opacity-80">실시간 남은 용돈 대시보드</h2>
                                        </div>
                                        <div class="text-3xl md:text-4xl font-extrabold mt-1.5 flex items-baseline gap-1">
                                            <span>{remainingBudget.toLocaleString()}</span>
                                            <span class="text-sm md:text-base font-normal">원 남음</span>
                                        </div>
                                    </div>
                                    <div class="text-left md:text-right w-full md:w-auto border-t md:border-t-0 border-slate-200/40 pt-3 md:pt-0">
                                        <div class="text-xs opacity-75 font-medium">총 예산: {budget.toLocaleString()}원</div>
                                        <div class="text-xs opacity-75 font-medium mt-0.5">이번 달 총 지출액: <span class="font-bold underline">{totalSpent.toLocaleString()}원</span></div>
                                    </div>
                                </div>

                                {/* 게이지 바 시각화 */}
                                <div class="w-full bg-slate-200/60 h-2.5 rounded-full mt-4 overflow-hidden">
                                    <div 
                                        class={`h-full rounded-full transition-all duration-500 ${isEmergency ? 'bg-rose-600' : 'bg-emerald-600'}`}
                                        style={{ width: `${Math.max(0, Math.min(100, remainingPercentage))}%` }}
                                    ></div>
                                </div>
                                <div class="flex justify-between items-center text-[11px] mt-1.5 opacity-75 font-medium">
                                    <span>남은 비율: {Math.max(0, Math.round(remainingPercentage))}%</span>
                                    {isEmergency && <span class="font-bold animate-pulse">📢 예산의 20% 미만입니다! 소비를 긴급 통제하세요!</span>}
                                </div>
                            </div>

                            {/* [기능 3] 카테고리별 지출 통계 및 원형 그래프 시각화 서브 섹션 */}
                            <div class="bg-white p-6 rounded-2xl shadow-xs border border-slate-100">
                                <h3 class="font-bold text-slate-900 mb-4 flex items-center gap-2">
                                    <i class="fa-solid fa-chart-pie text-indigo-600"></i> 카테고리별 소비 분석 리포트
                                </h3>
                                
                                {expenses.length === 0 ? (
                                    <div class="text-center py-8 text-slate-400 text-sm">
                                        <i class="fa-solid fa-chart-line text-2xl mb-2 block"></i>
                                        통계를 집계할 지출 내역이 없습니다.
                                    </div>
                                ) : (
                                    <div class="grid grid-cols-1 md:grid-cols-12 gap-6 items-center">
                                        
                                        {/* 경량 SVG 도넛 차트 시각화 컴포넌트 (3-2) */}
                                        <div class="md:col-span-4 flex justify-center">
                                            <div class="relative w-40 h-40">
                                                <svg viewBox="-1 -1 2 2" class="transform -rotate-90 w-full h-full">
                                                    {categoryStats.map((stat, index) => {
                                                        if (stat.amount === 0) return null;
                                                        const startPercent = cumulativePercent;
                                                        const itemPercent = stat.amount / totalSpent;
                                                        cumulativePercent += itemPercent;

                                                        const [startX, startY] = getCoordinatesForPercent(startPercent);
                                                        const [endX, endY] = getCoordinatesForPercent(cumulativePercent);
                                                        const largeArcFlag = itemPercent > 0.5 ? 1 : 0;
                                                        const pathData = `M ${startX} ${startY} A 1 1 0 ${largeArcFlag} 1 ${endX} ${endY} L 0 0`;
                                                        
                                                        return (
                                                            <path 
                                                                key={stat.meta.name} 
                                                                d={pathData} 
                                                                fill={stat.meta.color}
                                                                class="hover:opacity-85 transition-opacity cursor-pointer"
                                                            />
                                                        );
                                                    })}
                                                    {/* 가운데를 구멍 뚫어 도넛 디자인 적용 */}
                                                    <circle cx="0" cy="0" r="0.6" fill="#ffffff" />
                                                </svg>
                                                <div class="absolute inset-0 flex flex-col items-center justify-center pointer-events-none">
                                                    <span class="text-[10px] text-slate-400 font-medium">총 지출액</span>
                                                    <span class="text-xs font-bold text-slate-800">{totalSpent.toLocaleString()}원</span>
                                                </div>
                                            </div>
                                        </div>

                                        {/* 카테고리 텍스트 리스트 매핑 데이터 배열 조율 (3-1) */}
                                        <div class="md:col-span-8 space-y-3">
                                            {categoryStats.map((stat) => (
                                                <div key={stat.meta.name} class="flex items-center gap-3">
                                                    <span class={`w-2 h-2 rounded-full`} style={{ backgroundColor: stat.meta.color }}></span>
                                                    <div class="w-24 text-xs font-semibold text-slate-700 truncate">{stat.meta.name}</div>
                                                    <div class="flex-1 bg-slate-100 h-2 rounded-full overflow-hidden">
                                                        <div 
                                                            class="h-full rounded-full" 
                                                            style={{ width: `${stat.percentage}%`, backgroundColor: stat.meta.color }}
                                                        ></div>
                                                    </div>
                                                    <div class="w-20 text-right text-xs text-slate-500 font-medium">
                                                        <span class="text-slate-800 font-semibold">{stat.percentage}%</span> ({stat.amount.toLocaleString()}원)
                                                    </div>
                                                </div>
                                            ))}
                                        </div>
                                    </div>
                                )}
                            </div>

                            {/* [기능 2-3] 지출 내역 리스트 정렬 및 개별 항목 제어 필터 */}
                            <div class="bg-white p-6 rounded-2xl shadow-xs border border-slate-100">
                                <div class="flex flex-col sm:flex-row sm:items-center sm:justify-between gap-3 mb-4">
                                    <h3 class="font-bold text-slate-900 flex items-center gap-2">
                                        <i class="fa-solid fa-list-ul text-slate-500"></i> 스마트 지출 세부 원장
                                    </h3>
                                    
                                    {/* 정렬 필터 컨트롤러 */}
                                    <div class="flex items-center gap-1.5 self-end sm:self-auto">
                                        <button 
                                            onClick={() => setSortBy('latest')} 
                                            class={`px-3 py-1 rounded-lg text-xs font-medium transition-colors cursor-pointer ${sortBy === 'latest' ? 'bg-indigo-600 text-white' : 'bg-slate-100 text-slate-600 hover:bg-slate-200'}`}
                                        >
                                            ⏱ 최신 날짜순
                                        </button>
                                        <button 
                                            onClick={() => setSortBy('highest')} 
                                            class={`px-3 py-1 rounded-lg text-xs font-medium transition-colors cursor-pointer ${sortBy === 'highest' ? 'bg-indigo-600 text-white' : 'bg-slate-100 text-slate-600 hover:bg-slate-200'}`}
                                        >
                                            💰 금액 높은순
                                        </button>
                                    </div>
                                </div>

                                {/* 실제 내역 렌더링 배열 */}
                                {sortedExpenses.length === 0 ? (
                                    <div class="text-center py-12 text-slate-400 text-sm border-2 border-dashed border-slate-100 rounded-xl">
                                        <i class="fa-solid fa-receipt text-3xl mb-2 block text-slate-300"></i>
                                        등록된 가계부 내역이 하나도 없습니다.<br/>왼쪽 폼에서 첫 지출을 기록해 보세요!
                                    </div>
                                ) : (
                                    <div class="divide-y divide-slate-100 border border-slate-100 rounded-xl overflow-hidden bg-white">
                                        {sortedExpenses.map((item) => {
                                            // 매칭되는 카테고리 메타데이터 로드
                                            const catMeta = CATEGORIES.find(c => c.name === item.category) || { bg: 'bg-slate-50', text: 'text-slate-700', icon: 'fa-tag' };
                                            return (
                                                <div key={item.id} class="p-4 flex items-center justify-between hover:bg-slate-50/70 transition-colors group">
                                                    <div class="flex items-center gap-3 min-w-0">
                                                        {/* 카테고리별 맞춤 아이콘 */}
                                                        <div class={`w-9 h-9 rounded-xl flex items-center justify-center shrink-0 ${catMeta.bg} ${catMeta.text}`}>
                                                            <i class={`fa-solid ${catMeta.icon} text-sm`}></i>
                                                        </div>
                                                        <div class="min-w-0">
                                                            <div class="flex items-center gap-2">
                                                                <span class="font-bold text-sm text-slate-900 truncate">{item.memo}</span>
                                                                <span class="text-[10px] bg-slate-100 text-slate-500 px-1.5 py-0.5 rounded-sm font-medium shrink-0">{item.paymentMethod}</span>
                                                            </div>
                                                            <div class="text-[11px] text-slate-400 mt-0.5 flex items-center gap-1.5">
                                                                <span>{item.date}</span>
                                                                <span>•</span>
                                                                <span>{item.category}</span>
                                                            </div>
                                                        </div>
                                                    </div>
                                                    <div class="flex items-center gap-3 shrink-0 ml-4">
                                                        <span class="font-bold text-sm text-slate-900">-{item.amount.toLocaleString()}원</span>
                                                        
                                                        {/* 개별 제거 X 버튼 (2-3) */}
                                                        <button 
                                                            onClick={() => handleDeleteExpense(item.id)}
                                                            class="text-slate-300 hover:text-rose-600 p-1 rounded-md transition-colors opacity-100 lg:opacity-0 group-hover:opacity-100 cursor-pointer"
                                                            title="삭제"
                                                        >
                                                            <i class="fa-solid fa-xmark text-sm"></i>
                                                        </button>
                                                    </div>
                                                </div>
                                            );
                                        })}
                                    </div>
                                )}
                            </div>

                            {/* 바이브코딩 전략 안내 및 AI 협업 주석 정보 */}
                            <div class="bg-slate-900 text-slate-400 p-4 rounded-2xl text-xs space-y-1.5">
                                <div class="text-slate-200 font-semibold flex items-center gap-1.5 mb-1">
                                    <i class="fa-solid fa-robot text-indigo-400"></i> AI 에이전트 코딩 및 시스템 확장 아키텍처 규칙
                                </div>
                                <p>• <strong>데이터 바인딩 규칙:</strong> 입력 폼 이벤트 감지 후 React state 객체 정렬 상태로 실시간 대시보드 변수 오케스트레이션 수행.</p>
                                <p>• <strong>샌드박스 안정성:</strong> 본 단일 컴포넌트 환경은 Lovable 및 Bolt.new 배포를 지원하는 안전한 가상 DOM 트리거 훅 구조로 작성됨.</p>
                            </div>

                        </section>
                    </main>
                </div>
            );
        }

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<MoneyVibeApp />);
    </script>
</body>
</html>
