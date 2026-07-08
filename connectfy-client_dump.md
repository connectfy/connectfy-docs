# connectfy-client Source Dump

## File: connectfy-client/src/main.tsx
```typescript
import "./i18n.js";
import App from "./App.tsx";
import { Provider } from "react-redux";
import { store } from "@/store/store.ts";
import { createRoot } from "react-dom/client";
import { ThemeProvider } from "@/context/ThemeContext.tsx";
import { GoogleOAuthProvider } from "@react-oauth/google";
import { history } from "@/common/helpers/history.ts";
import { unstable_HistoryRouter as HistoryRouter } from "react-router-dom";
import { Toaster } from "react-hot-toast";
import { ContextMenuProvider } from "./context/ContextMenuContext.tsx";
import { UserProvider } from "./context/UserContext.tsx";
import { SockerProvider } from "./context/SocketContext.tsx";

const GOOGLE_CLIENT_ID = import.meta.env.VITE_GOOGLE_CLIENT_ID;

createRoot(document.getElementById("root")!).render(
  <GoogleOAuthProvider clientId={GOOGLE_CLIENT_ID}>
    <Provider store={store}>
      <HistoryRouter history={history as unknown as any}>
        <ThemeProvider>
          <UserProvider>
            <SockerProvider>
              <ContextMenuProvider>
                <Toaster
                  toastOptions={{ duration: 4000 }}
                  containerStyle={{ zIndex: 9999 }}
                />
                <App />
              </ContextMenuProvider>
            </SockerProvider>
          </UserProvider>
        </ThemeProvider>
      </HistoryRouter>
    </Provider>
  </GoogleOAuthProvider>,
);

```

## File: connectfy-client/src/global.d.ts
```typescript
declare module "*.module.css" {
  const classes: { [key: string]: string };
  export default classes;
}

```

## File: connectfy-client/src/vite-env.d.ts
```typescript
/// <reference types="vite/client" />

```

## File: connectfy-client/src/App.tsx
```typescript
import "@/styles/index.css";
import { Fragment, useEffect } from "react";
import routes from "@/routes/router";
import { initFlowbite } from "flowbite";
import "flag-icons/css/flag-icons.min.css";
import { useRoutes } from "react-router-dom";
import { useTranslation } from "react-i18next";
import { checkDeviceId } from "./common/utils/checkValues";
import { LANGUAGE, LOCAL_STORAGE_KEYS } from "@/common/enums/enums";
import { useGeneralSettings } from "./modules/settings/GeneralSettings/hooks/useGeneralSettings";
import { useNotificationSettings } from "./modules/settings/NotificationSettings/hooks/useNotificationSettings";
import { useAuthStore } from "./store/zustand/useAuthStore";
import GlobalModals from "./components/Modal/GlobalModals/GlobalModals";
import { useNotifications } from "./modules/notifications/hooks/useNotifications";
import { usePresenceHeartbeat } from "./common/hooks/usePresenceHeartbeat";

function App() {
  const { i18n } = useTranslation();
  const content = useRoutes(routes);
  const lang = localStorage.getItem(LOCAL_STORAGE_KEYS.LANG);
  const deviceId = checkDeviceId();
  const { access_token } = useAuthStore();

  const { data } = useGeneralSettings();
  useNotificationSettings();
  useNotifications();
  usePresenceHeartbeat();
  const userLang = data?.language;

  useEffect(() => {
    if (!access_token) {
      const availableLangs = Object.values(LANGUAGE);
      const validLang =
        lang && availableLangs.includes(lang as LANGUAGE)
          ? (lang as LANGUAGE)
          : LANGUAGE.EN;

      i18n.changeLanguage(validLang);
      localStorage.setItem(LOCAL_STORAGE_KEYS.LANG, validLang);
      return;
    }

    if (userLang && access_token) {
      i18n.changeLanguage(userLang);
      localStorage.removeItem(LOCAL_STORAGE_KEYS.LANG);
    }
  }, [lang, i18n, userLang, access_token]);

  useEffect(() => {
    checkDeviceId();
  }, [deviceId]);

  useEffect(() => {
    initFlowbite();
  }, []);

  return (
    <Fragment>
      {content}
      <GlobalModals />
    </Fragment>
  );
}

export default App;

```

## File: connectfy-client/src/components/Loader/Components/ComponentLoader.tsx
```typescript
import React, {
  Suspense,
  type ComponentType,
  type LazyExoticComponent,
} from "react";
import { LoadingSpinner } from "@/components/Spinner/Settings/LoadingSpinner";

type AnyComponent<P> = ComponentType<P> | LazyExoticComponent<ComponentType<P>>;

export default function ComponentLoader<P extends object>(
  Component: AnyComponent<P>,
  FallbackComponent: React.ReactNode = <LoadingSpinner />,
): ComponentType<P> {
  const Wrapped: React.FC<P> = (props) => {
    const C = Component as unknown as ComponentType<P>;
    return (
      <Suspense fallback={FallbackComponent}>
        <C {...props} />
      </Suspense>
    );
  };

  Wrapped.displayName = `ComponentLoader(${(Component as any).displayName || (Component as any).name || "Component"})`;
  return Wrapped;
}

```

## File: connectfy-client/src/components/Loader/Main/Loader.tsx
```typescript
// src/components/Loader.tsx
import React, {
  Suspense,
  type ComponentType,
  type LazyExoticComponent,
} from "react";
import Loading from "../../Loading/Loading.tsx";

type AnyComponent<P> = ComponentType<P> | LazyExoticComponent<ComponentType<P>>;

export default function Loader<P extends object>(
  Component: AnyComponent<P>
): ComponentType<P> {
  const Wrapped: React.FC<P> = (props) => {
    const C = Component as unknown as ComponentType<P>;
    return (
      <Suspense fallback={<Loading />}>
        <C {...props} />
      </Suspense>
    );
  };

  Wrapped.displayName = `Loader(${(Component as any).displayName || (Component as any).name || "Component"})`;
  return Wrapped;
}

```

## File: connectfy-client/src/components/Loading/Loading.tsx
```typescript
import MainIcon from "@/assets/icons/MainIcon";
import "./loading.style.css";
import { CSSProperties } from "react";

interface Props {
  description?: {
    title?: string;
    titleStyle?: CSSProperties;
  };
}

export default function MainSpinner({ description }: Props) {
  return (
    <div className="spinner-container">
      <div className="spinner-icon">
        <MainIcon styles={{ width: 100, height: 100 }} />
      </div>

      <div className="dots-loader">
        <span />
        <span />
        <span />
      </div>

      <div className="spinner-text">Connectfy</div>

      {description && (
        <span style={description.titleStyle}>{description.title}</span>
      )}
    </div>
  );
}

```

## File: connectfy-client/src/components/Sidebar/Auth/AuthSidebar.tsx
```typescript
import React from "react";
import MainIcon from "@/assets/icons/MainIcon.tsx";
import { useTranslation } from "react-i18next";

const AuthSidebar: React.FC = () => {
  const { t } = useTranslation();

  return (
    <div
      className="relative hidden w-full lg:flex lg:w-1/2 xl:w-7/12 overflow-hidden items-center justify-center p-16"
      style={{
        background: "var(--auth-sidebar-bg)",
      }}
    >
      {/* Arxa plandakı dekorativ parıltı (Glow) */}
      <div
        className="absolute top-[-20%] left-[-10%] w-[800px] h-[800px] rounded-full blur-[120px] opacity-20 pointer-events-none mix-blend-screen"
        style={{ background: "var(--auth-sidebar-glow)" }}
      ></div>

      <div className="relative z-10 w-full max-w-xl flex flex-col justify-center">
        {/* 1. Logo Hissəsi */}
        <div className="flex items-center gap-3 mb-10">
          <MainIcon
            className="size-10 rounded-xl flex items-center justify-center shadow-[0_0_20px_rgba(52,211,153,0.3)]"
            styles={{
              backgroundColor: "#34d399",
              color: "#ffffff",
              width: 45,
              height: 45,
              padding: 8,
            }}
          />
          <h2 className="text-3xl font-extrabold tracking-tighter text-(--text-(--primary-color))">
            Connectfy
          </h2>
        </div>

        {/* 2. Böyük Başlıq (Typography) */}
        <div className="space-y-8 mb-16">
          <h1 className="text-(--text-(--primary-color)) text-6xl xl:text-[5.5rem] font-[900] leading-[0.9] tracking-[-0.04em]">
            {t("common.auth_sidebar_headline_line1")}
            <br />
            {t("common.auth_sidebar_headline_line2")}
            <br />
            {t("common.auth_sidebar_headline_line3")}
          </h1>
          <p className="text-xl font-medium max-w-md leading-relaxed text-(--primary-color)">
            {t("common.auth_sidebar_description")}
          </p>
        </div>

        {/* 3. Glass Card & Custom Dot Logo */}
        <div
          className="w-full aspect-video rounded-[32px] border backdrop-blur-md overflow-hidden flex items-center justify-center relative group"
          style={{
            backgroundColor: "var(--auth-glass-bg)",
            borderColor: "var(--auth-glass-border)",
            boxShadow: "0 25px 50px -12px rgba(0, 0, 0, 0.5)",
          }}
        >
          {/* Kartın içindəki işıq effekti */}
          <div className="absolute inset-0 bg-gradient-to-tr from-white/5 to-transparent opacity-0 group-hover:opacity-100 transition-opacity duration-700"></div>

          {/* Nöqtəli Logo (Grain/Dot Grid) */}
          <div className="relative z-10 flex flex-col gap-3 opacity-80 group-hover:scale-110 transition-transform duration-500">
            {/* 3 sətirlik nöqtələr */}
            <div className="flex gap-3 justify-center">
              <div
                className="size-4 rounded-full opacity-30"
                style={{ backgroundColor: "var(--auth-sidebar-glow)" }}
              ></div>
              <div
                className="size-4 rounded-full opacity-60"
                style={{ backgroundColor: "var(--auth-sidebar-glow)" }}
              ></div>
              <div
                className="size-4 rounded-full opacity-30"
                style={{ backgroundColor: "var(--auth-sidebar-glow)" }}
              ></div>
            </div>
            <div className="flex gap-3 justify-center">
              <div
                className="size-4 rounded-full opacity-60"
                style={{ backgroundColor: "var(--auth-sidebar-glow)" }}
              ></div>
              <div
                className="size-4 rounded-full opacity-100 shadow-[0_0_15px_rgba(52,211,153,0.6)]"
                style={{ backgroundColor: "var(--auth-sidebar-glow)" }}
              ></div>
              <div
                className="size-4 rounded-full opacity-60"
                style={{ backgroundColor: "var(--auth-sidebar-glow)" }}
              ></div>
            </div>
            <div className="flex gap-3 justify-center">
              <div
                className="size-4 rounded-full opacity-30"
                style={{ backgroundColor: "var(--auth-sidebar-glow)" }}
              ></div>
              <div
                className="size-4 rounded-full opacity-60"
                style={{ backgroundColor: "var(--auth-sidebar-glow)" }}
              ></div>
              <div
                className="size-4 rounded-full opacity-30"
                style={{ backgroundColor: "var(--auth-sidebar-glow)" }}
              ></div>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
};

export default AuthSidebar;

```

## File: connectfy-client/src/components/Sidebar/UniqueSidebar/UniqueSidebar.tsx
```typescript
import { FC, Fragment, memo, useEffect, useState } from "react";
import { useLocation } from "react-router-dom";
import { ChevronRight, LucideProps } from "lucide-react";
import { ForwardRefExoticComponent, RefAttributes } from "react";
import { motion } from "framer-motion";

interface Subject {
  name: string;
  path: string;
  icon: ForwardRefExoticComponent<
    Omit<LucideProps, "ref"> & RefAttributes<SVGSVGElement>
  >;
  key: string;
  badge: number | null;
  onClick: () => void;
}

interface Props {
  title?: {
    name: string;
    icon: ForwardRefExoticComponent<
      Omit<LucideProps, "ref"> & RefAttributes<SVGSVGElement>
    >;
  };
  subjects: Subject[];
  mobile?: boolean;
}

const UniqueSidebar: FC<Props> = ({ title, subjects }) => {
  const location = useLocation();
  const [activeKey, setActiveKey] = useState<string | null>(null);

  useEffect(() => {
    const currentPath = location.pathname || "/";
    const matched = subjects.find((s) =>
      currentPath === s.path ? true : currentPath.startsWith(s.path),
    );
    if (matched) setActiveKey(matched.key);
    else setActiveKey(null);
  }, [location.pathname, subjects]);

  const handleClick = (s: Subject) => {
    setActiveKey(s.key);
    s.onClick();
  };

  return (
    <Fragment>
      <aside className="w-full mx-auto">
        <div
          className="w-full rounded-xl md:rounded-[16px] p-4 md:p-[18px] lg:p-5 shadow-[0_10px_40px_-10px_rgba(0,0,0,0.1)] 
             dark:shadow-[0_20px_50px_rgba(0,0,0,1)] 
             border border-black/3 dark:border-white/5"
        >
          <div className="flex items-center gap-3 pb-3 mb-3 border-b border-black/5 dark:border-white/5 select-none">
            {title && (
              <div
                className="relative overflow-hidden inline-flex items-center justify-center 
                    w-[40px] h-[40px] lg:w-[48px] lg:h-[48px] 
                    rounded-[12px] lg:rounded-[14px] 
                    bg-linear-to-br from-(--third-color) to-(--hover-bg)
                    shadow-[0_8px_20px_rgba(46,204,113,0.3)]"
              >
                <motion.div
                  initial={{ x: "-150%", skewX: -20 }}
                  animate={{ x: "150%" }}
                  transition={{
                    duration: 1.5,
                    repeat: Infinity,
                    ease: "easeInOut",
                    repeatDelay: 2,
                  }}
                  className="absolute inset-0 w-full h-full bg-linear-to-r from-transparent via-white/40 to-transparent z-1"
                />

                {/* ICON */}
                <title.icon className="w-5 h-5 lg:w-6 lg:h-6 text-white relative z-10" />
              </div>
            )}

            <h3 className="font-extrabold text-[18px] lg:text-[22px] text-(--text-primary)">
              {title?.name}
            </h3>
          </div>

          <ul className="list-none p-0 m-0 mt-3 flex flex-col gap-2.5">
            {subjects.map((s) => {
              const isActive = activeKey === s.key;

              return (
                <li
                  key={s.key}
                  onClick={() => handleClick(s)}
                  className={`flex items-center justify-between p-3 rounded-xl cursor-pointer transition-all duration-180 ease-out overflow-visible
                    ${
                      isActive
                        ? "bg-linear-to-br from-[rgba(46,204,113,0.12)] to-[rgba(46,204,113,0.04)] text-(--primary-color) shadow-[0_6px_18px_rgba(46,204,113,0.12)] translate-x-[6px]"
                        : "bg-transparent text-slate-500 hover:bg-[rgba(46,204,113,0.04)] hover:text-(--primary-color) hover:translate-x-[6px]"
                    }
                  `}
                >
                  <div className="flex items-center gap-3">
                    <div className="relative flex items-center justify-center w-[38px] h-[38px] lg:w-[44px] lg:h-[44px] rounded-[10px] bg-[#64748b]/6 overflow-visible">
                      <s.icon className="w-[18px] h-[18px] lg:w-5 lg:h-5" />

                      {s.badge !== null && s.badge > 0 && (
                        <span className="absolute -top-1 -right-1 lg:-top-1.5 lg:-right-1.5 bg-linear-to-br from-red-500 to-red-600 text-white text-[10px] font-bold px-[5px] py-px lg:px-1.5 lg:py-[2px] rounded-[10px] min-w-[20px] text-center">
                          {s.badge > 99 ? "99+" : s.badge}
                        </span>
                      )}
                    </div>
                    <span className="text-[14px] lg:text-[15px] font-semibold">
                      {s.name}
                    </span>
                  </div>
                  <ChevronRight className="w-5 h-5" />
                </li>
              );
            })}
          </ul>
        </div>
      </aside>
    </Fragment>
  );
};

export default memo(UniqueSidebar);

```

## File: connectfy-client/src/components/Sidebar/Desktop/DesktopSidebar.tsx
```typescript
import { Fragment, memo, useMemo } from "react";
import { motion } from "framer-motion";
import {
  MessageCircle,
  Users,
  Radio,
  UserCircle,
  Bell,
  Settings,
} from "lucide-react";
import { useLocation } from "react-router-dom";
import { ROUTER } from "@/common/constants/routet";
import { useTranslation } from "react-i18next";
import { getHomeRouteByStartup } from "@/common/utils/routes";
import { useGeneralSettings } from "@/modules/settings/GeneralSettings/hooks/useGeneralSettings";
import { useAppNavigation } from "@/hooks/useAppNavigation";
import { useContextMenu } from "@/hooks/useContextMenu";
import DesktopSidebarContextMenu from "@/components/ContextMenu/Sidebar/DesktopSidebarContextMenu";
import NoProfilePhotoIcon from "@/assets/icons/NoProfilePhotoIcon";
import { useUser } from "@/context/UserContext";
import { useCountUnreadQuery } from "@/modules/notifications/api/api";
import { useRequestsCountQuery } from "@/modules/users/MyFriends/api/api";

const NavItem = ({
  isActive,
  onClick,
  icon: Icon,
  name,
  badge,
  isProfile = false,
  avatarUrl = null,
  onContextMenu = null,
}: any) => (
  <div
    className={`relative group w-full h-[56px] flex items-center justify-center rounded-2xl cursor-pointer transition-all duration-300
      ${
        isActive
          ? "bg-linear-to-br from-[rgba(46,204,113,0.15)] to-[rgba(46,204,113,0.08)] text-(--third-color) shadow-[0_4px_16px_rgba(46,204,113,0.2)]"
          : "text-[#64748b] dark:text-[#94a3b8] hover:bg-[rgba(46,204,113,0.08)] dark:hover:bg-[rgba(46,204,113,0.12)] hover:-translate-y-0.5"
      }`}
    onClick={onClick}
    onContextMenu={onContextMenu}
  >
    {isActive && (
      <motion.div
        layoutId="sidebar-active-pill"
        className="absolute -left-3 w-1.5 h-9 bg-linear-to-b from-(--third-color) to-[#27ae60] rounded-r-full z-10"
        transition={{ type: "spring", stiffness: 300, damping: 30 }}
      />
    )}

    <div className="relative flex items-center justify-center">
      {isProfile ? (
        <div
          className={`relative size-10 rounded-full overflow-hidden border-2 ${isActive ? "border-(--primary-color)" : "border-[#64748b] dark:border-[#94a3b8]"}`}
        >
          {avatarUrl ? (
            <img
              src={avatarUrl}
              className="object-cover w-full h-full bg-(--skeleton-card-bg)"
              loading="eager"
              fetchPriority="high"
              decoding="async"
            />
          ) : (
            <div className="flex items-center justify-center w-full h-full">
              <NoProfilePhotoIcon />
            </div>
          )}
        </div>
      ) : (
        <Icon size={24} strokeWidth={2.2} />
      )}

      {badge && (
        <div className="absolute -top-2 -right-2.5 bg-red-500 text-white text-[10px] font-bold px-1.5 py-0.5 rounded-full min-w-[18px] text-center">
          {badge}
        </div>
      )}
    </div>

    <div className="absolute left-[85px] invisible group-hover:visible group-hover:left-[75px] opacity-0 group-hover:opacity-100 bg-[#1e293b] dark:bg-[#f8fafc] text-white dark:text-[#1e293b] px-3.5 py-2 rounded-xl text-xs font-semibold whitespace-nowrap transition-all duration-300 shadow-2xl z-99999">
      {name}
      <div className="absolute -left-1.5 top-1/2 -translate-y-1/2 border-y-[6px] border-y-transparent border-r-[6px] border-r-[#1e293b] dark:border-r-[#f8fafc]" />
    </div>
  </div>
);

const DesktopSidebar = () => {
  const { pathname } = useLocation();
  const { t } = useTranslation();
  const { navigate, isPending } = useAppNavigation();
  const { generalSettings } = useGeneralSettings();
  const { handleContextMenu } = useContextMenu();
  const { user } = useUser();
  const { data: countUnread } = useCountUnreadQuery();
  const { data: requestsCount } = useRequestsCountQuery();

  const menuItems = useMemo(
    () => [
      {
        key: "messenger",
        icon: MessageCircle,
        name: t("common.messenger"),
        path: ROUTER.MESSENGER.MAIN,
      },
      {
        key: "groups",
        icon: Users,
        name: t("common.groups"),
        path: ROUTER.GROUPS.MAIN,
      },
      {
        key: "channels",
        icon: Radio,
        name: t("common.channels"),
        path: ROUTER.CHANNELS.MAIN,
        badge: 3,
      },
      {
        key: "users",
        icon: UserCircle,
        name: t("common.users"),
        path: ROUTER.USERS.MAIN,
        badge:
          requestsCount && requestsCount > 99
            ? "99+"
            : requestsCount === 0
              ? undefined
              : requestsCount,
      },
      {
        key: "notifications",
        icon: Bell,
        name: t("common.notifications"),
        path: ROUTER.NOTIFICATIONS.MAIN,
        badge:
          countUnread && countUnread > 99
            ? "99+"
            : countUnread === 0
              ? undefined
              : countUnread,
      },
    ],
    [t, countUnread, requestsCount],
  );

  const getActiveKey = () => {
    if (pathname.startsWith(ROUTER.SETTINGS.MAIN)) return "settings";
    if (pathname.startsWith(ROUTER.PROFILE.MAIN)) return "profile";
    const matched = menuItems.find(
      (m) => pathname === m.path || pathname.startsWith(m.path),
    );
    return matched ? matched.key : null;
  };

  const activeItem = getActiveKey();

  return (
    <Fragment>
      <section
        id="sidebar"
        className="relative z-50 transition-opacity duration-300"
        style={{ opacity: isPending ? 0.7 : 1 }}
      >
        <div className="w-[85px] h-screen backdrop-blur-[20px] border-r border-black/5 dark:border-white/10 flex flex-col items-center py-5 shadow-lg bg-white/95 dark:bg-[#0a0f0d]/95">
          {/* Logo */}
          <div
            className="mb-9 cursor-pointer transition-transform hover:scale-110 hover:rotate-[5deg]"
            onClick={() =>
              navigate(getHomeRouteByStartup(generalSettings?.startupPage))
            }
          >
            <div className="relative w-12 h-12 bg-linear-to-br from-(--third-color) to-[#27ae60] rounded-2xl flex items-center justify-center shadow-[0_8px_24px_rgba(46,204,113,0.4)] overflow-hidden">
              {/* ✨ Parıltı animasiyası */}
              <div
                className="absolute pointer-events-none animate-logo-shine"
                style={{
                  top: "-50%",
                  left: "-50%",
                  width: "200%",
                  height: "200%",
                  background:
                    "linear-gradient(45deg, transparent, rgba(255,255,255,0.3), transparent)",
                }}
              />
              <svg
                viewBox="0 0 576 512"
                className="w-[28px] h-[28px] text-white fill-current relative z-10"
              >
                <path d="M416 192c0-88.4-93.1-160-208-160S0 103.6 0 192c0 34.3 14.1 65.9 38 92-13.4 30.2-35.5 54.2-35.8 54.5-2.2 2.3-2.8 5.7-1.5 8.7S4.8 352 8 352c36.6 0 66.9-12.3 88.7-25 32.2 15.7 70.3 25 111.3 25 114.9 0 208-71.6 208-160zm122 220c23.9-26 38-57.7 38-92 0-66.9-53.5-124.2-129.3-148.1.9 6.6 1.3 13.3 1.3 20.1 0 105.9-107.7 192-240 192-10.8 0-21.3-.8-31.7-1.9C207.8 439.6 281.8 480 368 480c41 0 79.1-9.2 111.3-25 21.8 12.7 52.1 25 88.7 25 3.2 0 6.1-1.9 7.3-4.8 1.3-2.9.7-6.3-1.5-8.7-.3-.3-22.4-24.2-35.8-54.5z" />
              </svg>
            </div>
          </div>

          {/* Main Navigation */}
          <div className="flex-1 flex flex-col gap-2 w-2/3 relative">
            {menuItems.map((item) => (
              <NavItem
                {...item}
                key={item.key}
                isActive={activeItem === item.key}
                onClick={() => navigate(item.path)}
              />
            ))}
          </div>

          {/* Settings & Profile */}
          <div className="w-2/3 flex flex-col gap-2 mb-2 pt-3 border-t border-black/5 dark:border-white/10">
            <NavItem
              isActive={activeItem === "settings"}
              onClick={() => navigate(ROUTER.SETTINGS.MAIN)}
              icon={Settings}
              name={t("common.settings")}
            />

            <div className="mt-1">
              <NavItem
                isActive={activeItem === "profile"}
                onClick={() => navigate(ROUTER.PROFILE.MAIN)}
                isProfile
                avatarUrl={user?.avatar?.url}
                name={t("common.my_profile")}
                onContextMenu={(e: any) =>
                  handleContextMenu(e, <DesktopSidebarContextMenu />)
                }
              />
            </div>
          </div>
        </div>
      </section>
    </Fragment>
  );
};

export default memo(DesktopSidebar);

```

## File: connectfy-client/src/components/Sidebar/Mobile/MobileSidebar.tsx
```typescript
import { memo, useMemo, useState, useEffect } from "react";
import { MessageCircle, Users, Radio, UserCircle, User } from "lucide-react";
import { useLocation } from "react-router-dom";
import { ROUTER } from "@/common/constants/routet";
import { useTranslation } from "react-i18next";
import { useUser } from "@/context/UserContext";
import Button from "@/components/ui/CustomButton/Button/Button";
import { useAppNavigation } from "@/hooks/useAppNavigation";
import NoProfilePhotoIcon from "@/assets/icons/NoProfilePhotoIcon";

const MobileSidebar = () => {
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();
  const location = useLocation();
  const { user } = useUser();

  const [activeItem, setActiveItem] = useState<string | null>(null);

  const menuItems = useMemo(
    () => [
      {
        key: "messenger",
        icon: MessageCircle,
        name: t("common.messenger"),
        path: ROUTER.MESSENGER.MAIN,
      },
      {
        key: "groups",
        icon: Users,
        name: t("common.groups"),
        path: ROUTER.GROUPS.MAIN,
      },
      {
        key: "channels",
        icon: Radio,
        name: t("common.channels"),
        path: ROUTER.CHANNELS.MAIN,
        badge: 3,
      },
      {
        key: "users",
        icon: UserCircle,
        name: t("common.users"),
        path: ROUTER.USERS.MAIN,
        badge: 12,
      },
      {
        key: "profile",
        icon: User,
        name: t("common.my_profile"),
        path: ROUTER.PROFILE.MAIN,
      },
    ],
    [t],
  );

  useEffect(() => {
    const currentPath = location.pathname || "/";
    const matched = menuItems.find(
      (m) => currentPath === m.path || currentPath.startsWith(m.path),
    );
    if (matched) {
      setActiveItem(matched.key);
    } else if (currentPath.startsWith(ROUTER.SETTINGS.MAIN)) {
      setActiveItem("profile");
    }
  }, [location.pathname, menuItems]);

  return (
    <div className="fixed z-50 w-[92%] h-16 max-w-lg -translate-x-1/2 border border-(--auth-glass-border) rounded-full bottom-5 left-1/2 bg-(--auth-glass-bg) backdrop-blur-md shadow-lg transition-all duration-300">
      <div className="grid h-full grid-cols-5 mx-auto">
        {menuItems.map((item) => {
          const isActive = activeItem === item.key;

          return (
            <Button
              key={item.key}
              onClick={() => navigate(item.path)}
              className={`inline-flex flex-col items-center justify-center px-2 group transition-colors duration-200 ${
                item.key === "messenger" ? "rounded-s-full" : ""
              } ${item.key === "profile" ? "rounded-e-full" : ""}`}
            >
              <div className="relative">
                {item.key === "profile" ? (
                  <div
                    className={`size-[28px] rounded-full overflow-hidden shadow-[0_8px_24px_var(--shadow-color)] border-2 ${isActive ? "border-(--primary-color)" : "border-(--text-color)"}`}
                  >
                    {user?.avatar?.url ? (
                      <img
                        src={user.avatar.url}
                        alt={`Profile picture of ${user?.username}`}
                        className="object-cover w-full h-full bg-(--skeleton-card-bg)"
                        loading="eager"
                        fetchPriority="high"
                        decoding="async"
                      />
                    ) : (
                      <div className="flex items-center justify-center w-full h-full bg-(--active-bg-2)">
                        <NoProfilePhotoIcon />
                      </div>
                    )}
                  </div>
                ) : (
                  <item.icon
                    size={22}
                    className={`transition-transform duration-200 group-active:scale-90 ${
                      isActive
                        ? "text-(--primary-color) scale-110"
                        : "text-(--muted-color) group-hover:text-(--primary-color)"
                    }`}
                  />
                )}
                {item.badge && (
                  <span className="absolute -top-2 -right-2 flex h-4 min-w-[16px] items-center justify-center rounded-full bg-red-600 px-1 text-[9px] font-bold text-white border border-(--bg-color)">
                    {item.badge}
                  </span>
                )}
              </div>
              {/* <span
                className={`text-[10px] mt-1 font-medium transition-colors ${
                  isActive ? "text-(--primary-color)" : "text-(--muted-color)"
                }`}
              >
                {item.name}
              </span> */}
            </Button>
          );
        })}
      </div>
    </div>
  );
};

export default memo(MobileSidebar);

```

## File: connectfy-client/src/components/Spinner/Spinner.tsx
```typescript
import { FC, CSSProperties } from "react";

interface Props {
  size?: number;
  style?: CSSProperties;
  className?: string;
}

const Spinner: FC<Props> = ({
  size = 20,
  style = { color: "#fff" },
  className = "",
}) => {
  return (
    <svg
      className={`animate-spin ${className}`}
      style={{ width: size, height: size, ...style }}
      viewBox="0 0 24 24"
      fill="none"
    >
      <circle
        className="opacity-25"
        cx="12"
        cy="12"
        r="10"
        stroke="currentColor"
        strokeWidth="4"
      />
      <path
        className="opacity-75"
        fill="currentColor"
        d="M4 12a8 8 0 018-8v3a5 5 0 00-5 5H4z"
      />
    </svg>
  );
};

export default Spinner;

```

## File: connectfy-client/src/components/Spinner/Settings/LoadingSpinner.tsx
```typescript
import "./loadingSpinner.style.css";

export const LoadingSpinner = () => (
  <div className="loading-spinner-container">
    <div className="loading-spinner"></div>
  </div>
);

```

## File: connectfy-client/src/components/Modal/index.tsx
```typescript
import React, { FC, ReactNode, useEffect } from "react";
import ReactDOM from "react-dom";

interface Props {
  open: boolean;
  onClose: () => void; // Function yerinə daha dəqiq tip
  children: ReactNode;
  onMouseDown?: (e: React.MouseEvent<HTMLDivElement>) => void;
}

const Modal: FC<Props> = ({ open, onClose, children, onMouseDown }) => {
  // ESC düyməsi ilə bağlama məntiqi
  useEffect(() => {
    if (!open) return;

    const onKey = (e: KeyboardEvent) => {
      if (e.key === "Escape") onClose();
    };

    document.addEventListener("keydown", onKey);
    // Modal açıq olduqda body-də scroll-u bağlamaq istəsəniz:
    document.body.style.overflow = "hidden";

    return () => {
      document.removeEventListener("keydown", onKey);
      document.body.style.overflow = "unset";
    };
  }, [open, onClose]);

  if (!open) return null;

  const handleMouseDown = (e: React.MouseEvent<HTMLDivElement>) => {
    // Yalnız overlay-ə kliklədikdə bağla
    if (e.target === e.currentTarget) {
      onClose();
    }
    if (onMouseDown) onMouseDown(e);
  };

  return ReactDOM.createPortal(
    <section
      role="dialog"
      aria-modal="true"
      onMouseDown={handleMouseDown}
      className="fixed inset-0 z-1000 flex h-full w-full items-center justify-center bg-black/60 backdrop-blur-xs transition-all"
    >
      {children}
    </section>,
    document.body,
  );
};

export default Modal;

```

## File: connectfy-client/src/components/Modal/AvatarModal/ShowAvatarModal/ShowAvatarModal.tsx
```typescript
import { FC, useState } from "react";
import { motion, AnimatePresence } from "framer-motion";
import {
  X,
  Copy,
  ZoomIn,
  ZoomOut,
  QrCode,
  ArrowLeft,
  UserCircle,
} from "lucide-react";
import { QRCodeCanvas } from "qrcode.react";
import Modal from "../../index";
import { useTranslation } from "react-i18next";
import Button from "@/components/ui/CustomButton/Button/Button";
import { snack } from "@/common/utils/snackManager";
import SharePopover from "../SharePopover/SharePopover";
import { useAvatarModalStore } from "@/store/zustand/useAvatarModalStore";
import TextTooltip from "@/components/Tooltip/TextTooltip";

interface IProps {
  open: boolean;
  onClose: () => void;
  avatarUrl: string;
  username?: string;
  userId?: string;
}

const ShowAvatarModal: FC<IProps> = ({
  open,
  onClose,
  avatarUrl,
  username,
  userId,
}) => {
  const { t } = useTranslation();

  const onOpenChangeModal = useAvatarModalStore(
    (state) => state.onOpenChangeModal,
  );
  const profileId = useAvatarModalStore((state) => state.profileId);
  const avatarObj = useAvatarModalStore((state) => state.avatarObj);

  const [scale, setScale] = useState(1);
  const [showQr, setShowQr] = useState(false);

  const profileUrl = `${window.location.origin}/users/profile/${userId}`;

  const handleZoomIn = () => setScale((prev) => Math.min(prev + 0.5, 3));
  const handleZoomOut = () => setScale((prev) => Math.max(prev - 0.5, 1));

  const handleCopyLink = () => {
    navigator.clipboard.writeText(profileUrl);
    snack.success(t("common.link_copied"));
  };

  const handleToggleQr = () => {
    setShowQr((prev) => !prev);
    setScale(1);
  };

  return (
    <Modal open={open} onClose={onClose}>
      <motion.div
        initial={{ opacity: 0, scale: 0.9 }}
        animate={{ opacity: 1, scale: 1 }}
        exit={{ opacity: 0, scale: 0.9 }}
        className="relative flex flex-col w-full max-w-lg overflow-hidden bg-(--auth-main-bg) rounded-3xl shadow-2xl"
      >
        {/* Header */}
        <div className="absolute top-0 left-0 right-0 z-20 flex items-center justify-between p-4 bg-linear-to-b from-black/40 to-transparent">
          <div className="flex items-center gap-2">
            {showQr && (
              <Button
                onClick={handleToggleQr}
                className="p-1.5 bg-white/10 hover:bg-white/20 backdrop-blur-md rounded-full text-white transition-all"
              >
                <ArrowLeft size={18} />
              </Button>
            )}
            <span className="font-medium text-white drop-shadow-md">
              {showQr
                ? t("common.qr_code")
                : username || t("common.profile_photo")}
            </span>
          </div>
          <Button
            onClick={onClose}
            className="p-2 transition-transform bg-white/10 hover:bg-white/20 backdrop-blur-md rounded-full text-white hover:scale-110"
          >
            <X size={20} />
          </Button>
        </div>

        {/* Image / QR Toggle Area */}
        <div className="relative flex items-center justify-center w-full aspect-square overflow-hidden group">
          <AnimatePresence mode="wait">
            {showQr ? (
              // ── QR Görünüşü ──
              <motion.div
                key="qr"
                initial={{ opacity: 0, scale: 0.85, rotateY: 90 }}
                animate={{ opacity: 1, scale: 1, rotateY: 0 }}
                exit={{ opacity: 0, scale: 0.85, rotateY: -90 }}
                transition={{ type: "spring", stiffness: 180, damping: 22 }}
                className="flex flex-col items-center justify-center gap-5 w-full h-full bg-(--auth-main-bg)"
              >
                {/* QR Wrapper - Instagram kimi ağ fon + avatar */}
                <div className="relative p-5 bg-white rounded-3xl shadow-xl">
                  <QRCodeCanvas
                    id="avatar-qr-code"
                    value={profileUrl}
                    size={200}
                    bgColor="#ffffff"
                    fgColor="#000000"
                    level="H"
                    imageSettings={{
                      src: avatarUrl,
                      x: undefined,
                      y: undefined,
                      height: 48,
                      width: 48,
                      excavate: true,
                    }}
                  />
                </div>

                {/* @username */}
                <div className="flex flex-col items-center gap-1">
                  <span className="text-lg font-bold text-(--text-primary)">
                    @{username}
                  </span>
                  <span className="text-xs text-(--text-secondary) opacity-60">
                    {t("common.scan_to_view_profile")}
                  </span>
                </div>
              </motion.div>
            ) : (
              // ── Şəkil Görünüşü ──
              <motion.div
                key="image"
                initial={{ opacity: 0 }}
                animate={{ opacity: 1 }}
                exit={{ opacity: 0 }}
                className="relative w-full h-full"
              >
                <motion.img
                  animate={{ scale }}
                  transition={{ type: "spring", stiffness: 200, damping: 25 }}
                  src={avatarUrl}
                  alt="Avatar"
                  className="object-contain w-full h-full"
                  onDoubleClick={scale > 1 ? () => setScale(1) : handleZoomIn}
                />
                {/* Zoom Controls */}
                <div className="absolute bottom-4 right-4 flex flex-col gap-2 opacity-0 group-hover:opacity-100 transition-opacity duration-300">
                  <Button
                    onClick={handleZoomIn}
                    className="p-2.5 bg-white/10 backdrop-blur-md border border-white/20 rounded-xl text-white hover:bg-white/20"
                  >
                    <ZoomIn size={18} />
                  </Button>
                  <Button
                    onClick={handleZoomOut}
                    className="p-2.5 bg-white/10 backdrop-blur-md border border-white/20 rounded-xl text-white hover:bg-white/20"
                  >
                    <ZoomOut size={18} />
                  </Button>
                </div>
              </motion.div>
            )}
          </AnimatePresence>
        </div>

        {/* Action Bar */}
        <div className="flex items-center justify-around p-5 bg-(--auth-main-bg) border-t border-(--auth-glass-border)">
          {/* 1. Share (Paylaş) */}
          <SharePopover
            profileUrl={profileUrl}
            username={username}
            text={`${t("common.check_out_my_profile")}: `}
          />

          {/* 2. Copy Link (Linki kopyala) */}
          <TextTooltip position="top" text={t("common.copy_link")}>
            <Button
              onClick={handleCopyLink}
              className="w-[90px] flex flex-col items-center gap-2 transition-colors text-(--text-secondary) hover:text-(--third-color) group min-w-0"
            >
              <div className="p-3 rounded-2xl bg-(--active-bg-2) group-hover:bg-(--active-bg) transition-colors shrink-0">
                <Copy size={22} />
              </div>
              {/* <span className="text-[11px] font-semibold tracking-wider truncate w-full text-center">
                {t("common.copy_link")}
              </span> */}
            </Button>
          </TextTooltip>

          {/* 3. QR Code (QR Kod) */}
          <TextTooltip position="top" text={t("common.qr_code")}>
            <Button
              onClick={handleToggleQr}
              className={`w-[90px] flex flex-col items-center gap-2 transition-colors group min-w-0 ${
                showQr
                  ? "text-(--third-color)"
                  : "text-(--text-secondary) hover:text-(--third-color)"
              }`}
            >
              <div
                className={`p-3 rounded-2xl transition-colors shrink-0 ${
                  showQr
                    ? "bg-(--active-bg)"
                    : "bg-(--active-bg-2) group-hover:bg-(--active-bg)"
                }`}
              >
                <QrCode size={22} />
              </div>
              {/* <span className="text-[11px] font-semibold tracking-wider truncate w-full text-center">
                {t("common.qr_code")}
              </span> */}
            </Button>
          </TextTooltip>

          {/* 4. Change Avatar (Dəyişdir/Tənzimlə) */}
          <TextTooltip position="top" text={t("common.change_avatar")}>
            <Button
              onClick={() => onOpenChangeModal(avatarObj, profileId)}
              className="w-[90px] flex flex-col items-center gap-2 transition-colors text-(--text-secondary) hover:text-(--third-color) group min-w-0"
            >
              <div className="p-3 rounded-2xl bg-(--active-bg-2) group-hover:bg-(--active-bg) transition-colors shrink-0">
                <UserCircle size={22} />
              </div>
              {/* <span className="text-[11px] font-semibold tracking-wider truncate w-full text-center">
                {t("common.change_avatar")}
              </span> */}
            </Button>
          </TextTooltip>
        </div>
      </motion.div>
    </Modal>
  );
};

export default ShowAvatarModal;

```

## File: connectfy-client/src/components/Modal/AvatarModal/UploadAvatarModal/UploadAvatarModal.tsx
```typescript
import Modal from "@/components/Modal";
import { FC, Fragment, useCallback, useState } from "react";
import { UploadCloud, X, ZoomIn, ZoomOut } from "lucide-react";
import { AnimatePresence, motion } from "framer-motion";
import Button from "@/components/ui/CustomButton/Button/Button";
import { useTranslation } from "react-i18next";
import { useUpdateAvatar } from "@/modules/profile/hooks/useUpdateAvatar";
import { ProfilePhotoUpdateAction } from "@/common/enums/enums";
import Cropper, { Area } from "react-easy-crop";
import { getCroppedImg } from "@/common/utils/cropImage";
import { snack } from "@/common/utils/snackManager";
import Spinner from "@/components/Spinner/Spinner";

interface IProps {
  open: boolean;
  onClose: () => void;
  profileId: string;
  isProfileLoading: boolean;
}

const UploadAvatarModal: FC<IProps> = ({
  open,
  onClose,
  profileId,
  isProfileLoading,
}) => {
  const { t } = useTranslation();
  const { handleAvatarUpload, validateFile } = useUpdateAvatar();

  const [selectedFileName, setSelectedFileName] = useState<string>("");
  const [imageSrc, setImageSrc] = useState<string | null>(null);
  const [crop, setCrop] = useState({ x: 0, y: 0 });
  const [zoom, setZoom] = useState(1);
  const [croppedAreaPixels, setCroppedAreaPixels] = useState<Area | null>(null);
  const [isProcessing, setIsProcessing] = useState(false);

  const onFileChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    if (e.target.files && e.target.files.length > 0) {
      const file = e.target.files[0];

      const error = validateFile(file);

      if (error) {
        snack.warning(error);
        return;
      }

      setSelectedFileName(file.name);

      const reader = new FileReader();
      reader.readAsDataURL(file);
      reader.onload = () => {
        setImageSrc(reader.result as string);
      };
    }
  };

  const onCropComplete = useCallback((_: Area, croppedPixels: Area) => {
    setCroppedAreaPixels(croppedPixels);
  }, []);

  const onSave = async () => {
    if (imageSrc && croppedAreaPixels) {
      try {
        setIsProcessing(true);
        // Kəsilmiş şəkli File formatında alırıq
        const croppedFile = await getCroppedImg(
          imageSrc,
          croppedAreaPixels,
          selectedFileName || "avatar.jpeg",
        );

        if (croppedFile) {
          handleAvatarUpload({
            _id: profileId,
            file: croppedFile,
            action: ProfilePhotoUpdateAction.Update,
          });

          handleClose();
        }
      } catch (e) {
        snack.error(t("error_messages.process_failed"));
      } finally {
        setIsProcessing(false);
      }
    }
  };

  const onDrop = (e: React.DragEvent<HTMLLabelElement>) => {
    e.preventDefault();
    e.stopPropagation();

    const file = e.dataTransfer.files?.[0];
    if (!file) return;

    const error = validateFile(file);
    if (error) {
      snack.warning(error);
      return;
    }

    setSelectedFileName(file.name);
    const reader = new FileReader();
    reader.readAsDataURL(file);
    reader.onload = () => setImageSrc(reader.result as string);
  };

  const onDragOver = (e: React.DragEvent<HTMLLabelElement>) => {
    e.preventDefault();
    e.stopPropagation();
  };

  const handleClose = () => {
    setImageSrc(null);
    setZoom(1);
    setCrop({ x: 0, y: 0 });
    setSelectedFileName("");
    onClose();
  };

  return (
    <Modal open={open} onClose={handleClose}>
      <motion.div
        initial={{ opacity: 0, scale: 0.95 }}
        animate={{ opacity: 1, scale: 1 }}
        exit={{ opacity: 0, scale: 0.95 }}
        transition={{ duration: 0.2, ease: "easeOut" }}
        className="relative w-full max-w-lg p-6 bg-(--auth-main-bg) border border-(--auth-glass-border) rounded-2xl shadow-xl"
      >
        {/* Header */}
        <div className="flex items-center justify-between mb-6">
          <h2 className="text-xl font-bold text-(--text-color)">
            {imageSrc
              ? t("common.crop_photo") || "Şəkli Kəs"
              : t("common.upload_new_photo")}
          </h2>
          <Button
            onClick={handleClose}
            className="p-2 transition-colors rounded-full text-(--muted-color) hover:bg-(--active-bg-2) hover:text-(--text-color)"
          >
            <X size={20} />
          </Button>
        </div>

        {isProfileLoading ? (
          <div className="flex items-center justify-center w-full h-10">
            <Spinner size={30} className="text-(--text-color)!" />
          </div>
        ) : (
          <Fragment>
            {/* Content Area */}
            <div className="w-full mb-6">
              <AnimatePresence mode="wait">
                {!imageSrc ? (
                  <motion.label
                    key="dropzone"
                    initial={{ opacity: 0 }}
                    animate={{ opacity: 1 }}
                    exit={{ opacity: 0 }}
                    onDrop={onDrop}
                    onDragOver={onDragOver}
                    htmlFor="dropzone-file"
                    className="flex flex-col items-center justify-center w-full h-64 transition-all duration-200 border-2 border-dashed rounded-xl cursor-pointer bg-(--input-bg) border-(--input-border) hover:bg-(--active-bg-2) hover:border-(--primary-color) group"
                  >
                    <div className="flex flex-col items-center justify-center pt-5 pb-6 text-center">
                      <UploadCloud
                        className="w-10 h-10 mb-4 transition-colors text-(--muted-color) group-hover:text-(--primary-color)"
                        strokeWidth={1.5}
                      />
                      <p className="mb-2 text-sm text-(--text-color)">
                        <span className="font-semibold text-(--primary-color)">
                          {t("common.click_to_upload")}
                        </span>{" "}
                        {t("common.or_drag")}
                      </p>
                      <p className="text-xs text-(--muted-color)">
                        {t("common.file_format")}
                      </p>
                    </div>
                    <input
                      id="dropzone-file"
                      type="file"
                      className="hidden"
                      accept="image/*"
                      onChange={onFileChange}
                    />
                  </motion.label>
                ) : (
                  <motion.div
                    key="cropper"
                    initial={{ opacity: 0 }}
                    animate={{ opacity: 1 }}
                    exit={{ opacity: 0 }}
                    className="flex flex-col items-center w-full gap-4"
                  >
                    {/* Cropper Container */}
                    <div className="relative w-full h-64 overflow-hidden rounded-xl bg-(--active-bg-2)">
                      <Cropper
                        image={imageSrc}
                        crop={crop}
                        zoom={zoom}
                        aspect={1}
                        cropShape="round"
                        showGrid={false}
                        onCropChange={setCrop}
                        onCropComplete={onCropComplete}
                        onZoomChange={setZoom}
                      />
                    </div>

                    {/* Zoom Controls */}
                    <div className="flex items-center w-full gap-3 px-2 mt-2">
                      <ZoomOut size={18} className="text-(--muted-color)" />
                      <input
                        type="range"
                        value={zoom}
                        min={1}
                        max={3}
                        step={0.1}
                        aria-label="Zoom"
                        onChange={(e) => setZoom(Number(e.target.value))}
                        className="
    w-full h-1.5 cursor-pointer rounded-lg
    appearance-none bg-(--input-border) 
    outline-none focus:outline-none focus:ring-0

    [&::-webkit-slider-thumb]:appearance-none 
    [&::-webkit-slider-thumb]:size-4 
    [&::-webkit-slider-thumb]:rounded-full 
    [&::-webkit-slider-thumb]:bg-(--primary-color)
    [&::-webkit-slider-thumb]:border-none
    [&::-webkit-slider-thumb]:transition-none
    
    hover:bg-(--input-border)
    active:bg-(--input-border)
    [&::-webkit-slider-thumb]:hover:bg-(--primary-color)
    [&::-webkit-slider-thumb]:active:bg-(--primary-color)
  "
                      />
                      <ZoomIn size={18} className="text-(--muted-color)" />
                    </div>
                  </motion.div>
                )}
              </AnimatePresence>
            </div>

            {/* Action Buttons */}
            <div className="flex items-center justify-center gap-3 mt-4">
              {imageSrc && (
                <Button
                  type="button"
                  onClick={() => {
                    setImageSrc(null);
                    setSelectedFileName("");
                  }}
                  className="px-6 py-2.5 font-medium transition-colors bg-transparent border rounded-lg text-(--text-color) border-(--input-border) hover:bg-(--input-bg) w-full"
                  title={t("common.back")}
                />
              )}

              {!imageSrc ? (
                <Button
                  type="button"
                  onClick={handleClose}
                  disabled={isProcessing}
                  className="px-6 py-2.5 font-medium transition-colors bg-transparent border rounded-lg text-(--text-color) border-(--input-border) hover:bg-(--input-bg) w-full"
                  title={t("common.cancel")}
                />
              ) : (
                <Button
                  disabled={isProcessing}
                  type="button"
                  onClick={onSave}
                  className="px-6 py-2.5 font-medium text-white transition-colors rounded-lg bg-(--primary-color) shadow-md w-full disabled:opacity-50"
                  title={t("common.save")}
                  isLoading={isProcessing}
                />
              )}
            </div>
          </Fragment>
        )}
      </motion.div>
    </Modal>
  );
};

export default UploadAvatarModal;

```

## File: connectfy-client/src/components/Modal/AvatarModal/ChangeDefaultAvatarModal/ChangeDefaultAvatarModal.tsx
```typescript
import {
  IDefaultAvatar,
  IEditDefaultAvatar,
} from "@/modules/profile/types/types";
import Modal from "../.."; // Yolunu layihənə uyğun yoxla
import { useTranslation } from "react-i18next";
import { motion } from "framer-motion";
import NativeSelect from "@/components/ui/Select/NativeSelect/NativeSelect";
import { AvatarFormats } from "@/common/enums/enums";
import { useFormik } from "formik";
import { useUpdateDefaultAvatarMutation } from "@/modules/profile/api/api";
import { useErrors } from "@/hooks/useErrors";
import Button from "@/components/ui/CustomButton/Button/Button";
import ToggleSlider from "@/components/ui/CustomCheckbox/ToggleSlider/ToggleSlider";
import { X } from "lucide-react";
import useFormDisabled from "@/hooks/useFormDisabled";
import { snack } from "@/common/utils/snackManager";

interface IProps {
  open: boolean;
  onClose: () => void;
  defaultAvatar?: IDefaultAvatar;
  profileId: string;
}

const ChangeDefaultAvatarModal = ({
  open,
  onClose,
  defaultAvatar,
  profileId,
}: IProps) => {
  const { t } = useTranslation();
  const { showResponseErrors } = useErrors();

  const [updateDefaultAvatar, { isLoading }] = useUpdateDefaultAvatarMutation();

  const formatOptions = Object.values(AvatarFormats).map((format) => ({
    label: format.charAt(0).toUpperCase() + format.slice(1),
    value: format,
  }));

  const initialState: IEditDefaultAvatar = {
    _id: profileId,
    useDefaultAvatar: false,
    format: defaultAvatar?.format || AvatarFormats.Adventurer,
  };

  const formik = useFormik({
    initialValues: initialState,
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    onSubmit: async (values) => {
      try {
        await updateDefaultAvatar(values).unwrap();
        snack.success(t("user_messages.information_updated"));
        onClose();
      } catch (error) {
        showResponseErrors(error);
      }
    },
  });

  const isFormDisabled = useFormDisabled({
    formik,
    loading: isLoading,
    validationRules: [(values) => values.format !== defaultAvatar?.format],
  });

  if (!open || !defaultAvatar) return null;

  const { values, setFieldValue, handleSubmit } = formik;

  return (
    <Modal open={open} onClose={onClose}>
      <motion.div
        initial={{ opacity: 0, scale: 0.95, y: 10 }}
        animate={{ opacity: 1, scale: 1, y: 0 }}
        exit={{ opacity: 0, scale: 0.95, y: 10 }}
        transition={{ duration: 0.2, ease: "easeOut" }}
        className="flex flex-col w-full max-w-md p-6 bg-(--auth-main-bg) rounded-2xl border border-(--auth-glass-border) shadow-xl"
      >
        {/* Header */}
        <div className="flex items-center justify-between mb-6">
          <h2 className="text-xl font-bold text-(--text-color)">
            {t("common.change_default_avatar")}
          </h2>
          <Button
            type="button"
            onClick={onClose}
            className="p-2 transition-colors rounded-full text-(--muted-color) hover:bg-(--active-bg-2) hover:text-(--text-color)"
          >
            <X size={20} />
          </Button>
        </div>

        <form onSubmit={handleSubmit} className="flex flex-col w-full gap-6">
          {/* Live Preview Container */}
          <div className="flex justify-center w-full">
            <div className="relative flex items-center justify-center overflow-hidden border-4 rounded-full size-32 bg-(--active-bg-2) border-(--primary-color) shadow-[0_8px_24px_var(--shadow-color)]">
              <img
                src={`https://api.dicebear.com/9.x/${values.format}/svg?seed=${encodeURIComponent(defaultAvatar.seed)}`}
                alt="Avatar Preview"
                className="block object-cover w-full h-full"
                loading="eager"
                fetchPriority="high"
              />
            </div>
          </div>

          {/* Controls Container */}
          <div className="flex flex-col gap-4">
            {/* Format Selector */}
            <NativeSelect
              title={t("common.format")}
              inputSize="large"
              options={formatOptions}
              value={values.format} // FIX: Canlı baxış üçün formik state-ə bağlandı
              disabled={isLoading}
              onChange={(value) => setFieldValue("format", value)}
            />

            {/* Toggle Action */}
            <div className="flex items-center justify-between p-4 rounded-xl bg-(--active-bg-2) border border-(--auth-glass-border)">
              <span className="text-sm font-medium select-none text-(--text-color)">
                {t("common.use_as_default_avatar")}
              </span>
              <ToggleSlider
                checked={values.useDefaultAvatar}
                onClick={() =>
                  setFieldValue("useDefaultAvatar", !values.useDefaultAvatar)
                }
              />
            </div>
          </div>

          {/* Action Buttons */}
          <div className="flex items-center gap-3 mt-2">
            <Button
              type="button"
              onClick={onClose}
              disabled={isLoading}
              className="px-6 py-2.5 font-medium transition-colors bg-transparent border rounded-lg text-(--text-color) border-(--input-border) w-full"
              title={t("common.cancel")}
            />
            <Button
              type="submit"
              disabled={isFormDisabled}
              isLoading={isLoading}
              className="px-6 py-2.5 font-medium text-white transition-colors rounded-lg bg-(--primary-color) shadow-md w-full"
              title={t("common.save")}
            />
          </div>
        </form>
      </motion.div>
    </Modal>
  );
};

export default ChangeDefaultAvatarModal;

```

## File: connectfy-client/src/components/Modal/AvatarModal/SharePopover/SharePopover.tsx
```typescript
import { FC, useRef } from "react";
import { Share2 } from "lucide-react";
import { motion, AnimatePresence } from "framer-motion";
import { useClickAway } from "react-use";
import { useShare } from "@/hooks/useShare";
import Button from "@/components/ui/CustomButton/Button/Button";
import { useTranslation } from "react-i18next";
import TextTooltip from "@/components/Tooltip/TextTooltip";

interface IProps {
  profileUrl: string;
  text: string;
  username?: string;
}

const SharePopover: FC<IProps> = ({ profileUrl, username, text }) => {
  const { t } = useTranslation();
  const ref = useRef<HTMLDivElement>(null);

  const { isOpen, setIsOpen, shareLinks, handleOpen } = useShare({
    url: profileUrl,
    text: text,
    title: username,
  });

  useClickAway(ref, () => setIsOpen(false));

  return (
    <div className="relative" ref={ref}>
      <TextTooltip position="top" text={t("common.share")}>
        <Button
          onClick={() => setIsOpen((prev) => !prev)}
          className="w-[90px] flex flex-col items-center gap-1.5 transition-colors text-(--text-secondary) hover:text-(--third-color) group"
        >
          <div className="p-3 rounded-2xl bg-(--active-bg-2) group-hover:bg-(--active-bg) transition-colors">
            <Share2 size={22} />
          </div>
          {/* <span className="text-[11px] font-semibold tracking-wider">
            {t("common.share")}
          </span> */}
        </Button>
      </TextTooltip>

      <AnimatePresence>
        {isOpen && (
          <motion.div
            initial={{ opacity: 0, y: 8, scale: 0.95 }}
            animate={{ opacity: 1, y: 0, scale: 1 }}
            exit={{ opacity: 0, y: 8, scale: 0.95 }}
            transition={{ duration: 0.15 }}
            className="absolute bottom-full mb-3 left-1/2 -translate-x-1/2 z-50
                       bg-(--auth-main-bg) border border-(--auth-glass-border)
                       rounded-2xl shadow-xl p-2 flex flex-col gap-1 min-w-[160px]"
          >
            {shareLinks.map(({ label, icon, href }) => (
              <Button
                key={label}
                onClick={() => handleOpen(href)}
                className="flex items-center gap-3 px-3 py-2.5 rounded-xl
                           hover:bg-(--active-bg) transition-colors text-sm
                           text-(--text-secondary) hover:text-(--third-color) w-full text-left font-semibold"
              >
                {icon}
                {label}
              </Button>
            ))}
          </motion.div>
        )}
      </AnimatePresence>
    </div>
  );
};

export default SharePopover;

```

## File: connectfy-client/src/components/Modal/AvatarModal/ChangeAvatarModal/ChangeAvatartModal.tsx
```typescript
import Modal from "@/components/Modal";
import { FC } from "react";
import { ImagePlus, UserCircle, Trash2 } from "lucide-react";
import { motion } from "framer-motion";
import { IAvatar } from "@/modules/profile/types/types";
import Button from "@/components/ui/CustomButton/Button/Button";
import { useTranslation } from "react-i18next";
import { ProfilePhotoUpdateAction } from "@/common/enums/enums";
import { useUpdateAvatar } from "@/modules/profile/hooks/useUpdateAvatar";
import { useAvatarModalStore } from "@/store/zustand/useAvatarModalStore";
import { useUser } from "@/context/UserContext";
import Spinner from "@/components/Spinner/Spinner";

interface IProps {
  open: boolean;
  onClose: () => void;
  avatar?: IAvatar | null;
  profileId: string;
  isProfileLoading: boolean;
}

const ChangeAvatarModal: FC<IProps> = ({
  open,
  onClose,
  avatar,
  profileId,
  isProfileLoading,
}) => {
  const { t } = useTranslation();
  const { user } = useUser();

  const onOpenUploadModal = useAvatarModalStore(
    (state) => state.onOpenUploadModal,
  );
  const onOpenSetDefaultModal = useAvatarModalStore(
    (state) => state.onOpenSetDefaultModal,
  );

  const isSetDefaultEnabled = avatar?.isCustom === true || !avatar;
  const isRemoveEnabled = !!avatar;

  const { handleAvatarUpload, isLoading } = useUpdateAvatar();

  const handleSetDefault = () => {
    handleAvatarUpload({
      _id: profileId,
      action: ProfilePhotoUpdateAction.SetDefault,
    });
    onClose();
  };

  const handleRemove = () => {
    handleAvatarUpload({
      _id: profileId,
      action: ProfilePhotoUpdateAction.Remove,
    });
    onClose();
  };

  const handleUpdateClick = () => {
    onClose();
    setTimeout(() => {
      onOpenUploadModal();
    }, 150);
  };

  const handleSetDefaultClick = () => {
    onClose();
    setTimeout(() => {
      onOpenSetDefaultModal(profileId, user?.defaultAvatar);
    }, 150);
  };

  return (
    <Modal open={open} onClose={onClose}>
      <motion.div
        initial={{ opacity: 0, scale: 0.95, y: 10 }}
        animate={{ opacity: 1, scale: 1, y: 0 }}
        exit={{ opacity: 0, scale: 0.95, y: 10 }}
        transition={{ duration: 0.2, ease: "easeOut" }}
        className="flex flex-col w-full max-w-sm gap-2 p-6 bg-(--auth-main-bg) rounded-2xl border border-(--auth-glass-border) shadow-xl"
      >
        <h2 className="mb-2 text-xl font-bold text-(--text-color)">
          {t("common.profile_photo")}
        </h2>

        {isProfileLoading ? (
          <div className="flex items-center justify-center w-full h-10">
            <Spinner size={30} className="text-(--text-color)!" />
          </div>
        ) : (
          <div className="flex flex-col w-full gap-1">
            {/* 1. Update Profile Photo */}
            <Button
              onClick={handleUpdateClick}
              disabled={isLoading}
              className="flex items-center w-full gap-3 p-3 font-medium transition-colors rounded-xl text-(--text-color) hover:bg-(--active-bg-2)"
            >
              <div className="flex items-center justify-center w-10 h-10 rounded-full bg-(--icon-green-bg) text-(--icon-green-text)">
                <ImagePlus size={20} />
              </div>
              {t("common.update_profile_photo")}
            </Button>

            {/* 2. Set Default Avatar */}
            <Button
              onClick={handleSetDefault}
              disabled={!isSetDefaultEnabled || isLoading}
              className="flex items-center w-full gap-3 p-3 font-medium transition-colors rounded-xl text-(--text-color) hover:bg-(--active-bg-2)"
            >
              <div className="flex items-center justify-center w-10 h-10 rounded-full bg-(--icon-blue-bg) text-(--icon-blue-text)">
                <UserCircle size={20} />
              </div>
              {t("common.set_default_avatar")}
            </Button>

            {/* 3. Change Default Avatar */}
            <Button
              onClick={handleSetDefaultClick}
              disabled={isLoading}
              className="flex items-center w-full gap-3 p-3 font-medium transition-colors rounded-xl text-(--text-color) hover:bg-(--active-bg-2)"
            >
              <div className="flex items-center justify-center w-10 h-10 rounded-full bg-(--icon-orange-bg) text-(--icon-orange-text)">
                <UserCircle size={20} />
              </div>
              {t("common.change_default_avatar")}
            </Button>

            {/* 4. Remove Profile Photo */}
            <Button
              onClick={handleRemove}
              disabled={!isRemoveEnabled || isLoading}
              className="flex items-center w-full gap-3 p-3 font-medium transition-colors rounded-xl text-(--error-color) hover:bg-red-500/10 dark:hover:bg-red-500/20"
            >
              <div className="flex items-center justify-center w-10 h-10 rounded-full bg-red-500/10 text-(--error-color)">
                <Trash2 size={20} />
              </div>
              {t("common.remove_profile_photo")}
            </Button>
          </div>
        )}
      </motion.div>
    </Modal>
  );
};

export default ChangeAvatarModal;

```

## File: connectfy-client/src/components/Modal/SelectionModal/SelectionModal.tsx
```typescript
import { Check, LucideProps, X } from "lucide-react";
import {
  FC,
  ForwardRefExoticComponent,
  Fragment,
  RefAttributes,
  useEffect,
} from "react";
import Modal from "..";
import Button from "@/components/ui/CustomButton/Button/Button";

interface Selections {
  name: string;
  title?: string;
  onClick: () => void;
  icon: ForwardRefExoticComponent<
    Omit<LucideProps, "ref"> & RefAttributes<SVGSVGElement>
  >;
  key: string;
}

interface Props {
  title: string;
  selections: Selections[];
  open: boolean;
  onClose: () => void;
  activeKey: string;
}

const SelectionModal: FC<Props> = ({
  title,
  selections,
  open,
  onClose,
  activeKey,
}) => {
  useEffect(() => {
    if (open) {
      document.body.style.overflow = "hidden";
    } else {
      document.body.style.overflow = "unset";
    }
    return () => {
      document.body.style.overflow = "unset";
    };
  }, [open]);

  const handleOverlayPointerDown = (e: React.MouseEvent<HTMLDivElement>) => {
    if (e.target === e.currentTarget) {
      onClose();
    }
  };

  return (
    <Fragment>
      <Modal
        open={open}
        onClose={onClose}
        onMouseDown={handleOverlayPointerDown}
      >
        <div
          className="fixed top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 w-[90%] max-w-[480px] max-h-[80vh] bg-(--bg-color) rounded-xl shadow-(--card-shadow) z-1000000 flex flex-col animate-in fade-in zoom-in-95 duration-300 sm:w-[95%] sm:max-h-[85vh]"
          style={{ animation: "slideUp 0.3s cubic-bezier(0.16, 1, 0.3, 1)" }}
        >
          {/* Header */}
          <div className="flex items-center justify-between p-5 sm:p-4 border-b border-(--border-color)">
            <h3 className="m-0 text-[18px] sm:text-[16px] font-semibold text-(--text-color)">
              {title}
            </h3>
            <Button
              className="w-8 h-8 flex items-center justify-center rounded-md text-(--muted-color) hover:bg-(--secondary-color) hover:text-(--text-color) active:scale-95 transition-all"
              onClick={onClose}
              aria-label="Close modal"
              icon={<X size={18} />}
            />
          </div>

          {/* Content */}
          <div className="p-3 overflow-y-auto flex-1 scrollbar-thin scrollbar-thumb-[var(--muted-color)] scrollbar-track-transparent">
            {selections.map((selection) => {
              const Icon = selection.icon;
              const isActive = selection.key === activeKey;

              return (
                <Button
                  key={selection.key}
                  className={`flex items-center gap-3 w-full p-3 mb-3 rounded-lg text-left transition-all duration-200 active:scale-[0.98] last:mb-0
                    ${
                      isActive
                        ? "bg-(--active-bg) shadow-(--active-shadow)"
                        : "bg-transparent hover:bg-black/5 dark:hover:bg-(--secondary-color)"
                    }`}
                  onClick={() => {
                    selection.onClick();
                    onClose();
                  }}
                >
                  {/* Icon Wrapper */}
                  <div
                    className={`flex items-center justify-center w-10 h-10 sm:w-9 sm:h-9 shrink-0 rounded-lg transition-all duration-200
                    ${
                      isActive
                        ? "bg-(--primary-color) text-white"
                        : "bg-(--secondary-color) text-(--text-color)"
                    }`}
                  >
                    <Icon size={20} />
                  </div>

                  {/* Text Content */}
                  <div className="flex-1 flex flex-col gap-[2px]">
                    <div className="text-[15px] sm:text-[14px] font-medium text-(--text-color) leading-[1.4]">
                      {selection.name}
                    </div>
                    {selection.title && (
                      <div className="text-[13px] sm:text-[12px] text-(--muted-color) leading-[1.4]">
                        {selection.title}
                      </div>
                    )}
                  </div>

                  {/* Check Icon */}
                  {isActive && (
                    <div className="flex items-center justify-center w-6 h-6 shrink-0 rounded-full bg-(--primary-color) text-white">
                      <Check size={18} />
                    </div>
                  )}
                </Button>
              );
            })}
          </div>
        </div>
      </Modal>
    </Fragment>
  );
};

export default SelectionModal;

```

## File: connectfy-client/src/components/Modal/ActionConfirmModal/ActionConfirmModal.tsx
```typescript
import { FC, ForwardRefExoticComponent, ReactNode, RefAttributes } from "react";
import Modal from "..";
import Button from "@/components/ui/CustomButton/Button/Button";
import { LucideProps } from "lucide-react";

interface Props {
  open: boolean;
  onClose: () => void;
  onConfirm: () => void;
  onCancel: () => void;
  header: { title: string };
  children?: string | ReactNode;
  cancelBtn: { title: string };
  confirmBtn: { title: string; color: string };
  icon?: {
    content: ForwardRefExoticComponent<
      Omit<LucideProps, "ref"> & RefAttributes<SVGSVGElement>
    >;
    color: string;
  };
  onMouseDown?: (e: React.MouseEvent<HTMLDivElement>) => void;
  isLoading: boolean;
}

const ActionConfirmModal: FC<Props> = ({
  open,
  onClose,
  onConfirm,
  onCancel,
  header,
  children,
  cancelBtn,
  confirmBtn,
  icon,
  onMouseDown,
  isLoading,
}) => {
  return (
    <Modal open={open} onClose={onClose} onMouseDown={onMouseDown}>
      <div
        className="w-full max-w-[420px] overflow-hidden rounded-2xl bg-(--card-bg) shadow-2xl animate-slide-up"
        // style={{
        //   position: "fixed",
        //   top: "50%",
        //   left: "50%",
        //   transform: "translate(-50%, -50%)",
        // }}
      >
        <div className="flex flex-col items-center p-8">
          {/* Icon with Soft Glow */}
          {icon && (
            <div className="relative mb-6">
              <div
                className="absolute inset-0 scale-150 blur-2xl opacity-20 rounded-full"
                style={{ backgroundColor: icon.color }}
              />
              <div
                className="relative flex items-center justify-center rounded-full p-4 ring-8 ring-opacity-10"
                style={
                  {
                    backgroundColor: `${icon.color}15`,
                    color: icon.color,
                    boxShadow: `inset 0 0 0 1px ${icon.color}30`,
                    "--tw-ring-color": icon.color,
                  } as any
                }
              >
                <icon.content size={32} strokeWidth={2.5} />
              </div>
            </div>
          )}

          {/* Header */}
          <h2 className="mb-2 text-center text-2xl font-bold tracking-tight text-(--text-color)">
            {header.title}
          </h2>

          {/* Body Content */}
          <div className="w-full">
            {typeof children === "string" ? (
              <p className="text-center text-[15px] leading-relaxed text-(--muted-color)">
                {children}
              </p>
            ) : (
              children
            )}
          </div>

          {/* Actions */}
          <div className="mt-8 flex w-full gap-3">
            <Button
              type="button"
              disabled={isLoading}
              onClick={() => onCancel()}
              className="flex-1 rounded-xl border border-(--border-color) bg-transparent py-3 text-sm font-semibold text-(--text-color) transition-all hover:bg-(--disabled-bg) active:scale-95"
              title={cancelBtn.title}
            />
            <Button
              type="button"
              disabled={isLoading}
              onClick={() => onConfirm()}
              style={{ backgroundColor: confirmBtn.color }}
              className="flex flex-1 items-center justify-center rounded-xl py-3 text-sm font-semibold text-white shadow-lg shadow-opacity-20 transition-all hover:brightness-110 active:scale-95"
              title={confirmBtn.title}
              isLoading={isLoading}
            />
          </div>
        </div>
      </div>
    </Modal>
  );
};

export default ActionConfirmModal;

```

## File: connectfy-client/src/components/Modal/CountryCodeModal/CountryCodeModal.tsx
```typescript
import { type FC, useMemo, useState } from "react";
import { motion, AnimatePresence } from "framer-motion";
import Input from "@/components/ui/CustomInput/Input/Input.tsx";
import { useTranslation } from "react-i18next";
import { ICountry } from "@/common/interfaces/interfaces";
import { COUNTRIES } from "@/common/constants/constants";
import Button from "@/components/ui/CustomButton/Button/Button";
import Modal from "..";

interface Props {
  open: boolean;
  onClose: () => void;
  onSelect?: (country: ICountry) => void;
  initialSelectedKey?: string;
}

export const CountryCodeModal: FC<Props> = ({
  open,
  onClose,
  onSelect,
  initialSelectedKey,
}) => {
  const { t } = useTranslation();
  const [query, setQuery] = useState<string>("");

  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase();
    if (!q) return COUNTRIES;
    return COUNTRIES.filter(
      (c) =>
        c.name.toLowerCase().includes(q) ||
        c.code.toLowerCase().includes(q) ||
        c.key.toLowerCase().includes(q),
    );
  }, [query]);

  const handleSelect = (country: ICountry) => {
    onSelect?.(country);
    onClose();
  };

  return (
    <Modal open={open} onClose={onClose}>
      <AnimatePresence>
        {open && (
          <motion.div
            initial={{ opacity: 0, y: 40 }}
            animate={{ opacity: 1, y: 0 }}
            exit={{ opacity: 0, y: 40 }}
            transition={{ duration: 0.25, ease: "easeOut" }}
            className="w-[520px] max-w-[96%] max-h-[84vh] overflow-hidden rounded-xl bg-(--bg-color) shadow-(--card-shadow) flex flex-col"
            role="dialog"
            aria-modal="true"
            aria-labelledby="country-code-modal"
            onClick={(e) => e.stopPropagation()}
          >
            {/* Header */}
            <div className="flex items-center justify-between p-4 border-b border-black/5 dark:border-white/5">
              <h3
                id="country-code-modal"
                className="m-0 text-base font-semibold text-(--text-color)"
              >
                {t("common.select_country")}
              </h3>
              <Button
                onClick={onClose}
                aria-label="close"
                className="p-1.5 rounded-lg text-(--muted-color) hover:bg-(--active-bg-2) hover:text-(--text-color) transition-all duration-200"
              >
                <svg
                  xmlns="http://www.w3.org/2000/svg"
                  width="16"
                  height="16"
                  viewBox="0 0 24 24"
                  fill="none"
                  stroke="currentColor"
                  strokeWidth="2"
                  strokeLinecap="round"
                  strokeLinejoin="round"
                >
                  <line x1="18" y1="6" x2="6" y2="18" />
                  <line x1="6" y1="6" x2="18" y2="18" />
                </svg>
              </Button>
            </div>

            {/* Content */}
            <div className="p-4 pt-0 flex flex-col overflow-hidden">
              <div className="py-4">
                <Input
                  title={t("common.search_country")}
                  value={query}
                  inputSize="medium"
                  onChange={(e) => setQuery(e.target.value)}
                />
              </div>

              {/* Country List */}
              <div
                className="overflow-y-auto space-y-1 pr-1 scrollbar-thin scrollbar-thumb-[var(--muted-color)]"
                style={{ maxHeight: "calc(84vh - 160px)" }}
                role="list"
              >
                {filtered.map((c) => {
                  const isSelected = initialSelectedKey === c.key;
                  return (
                    <Button
                      key={c.key}
                      type="button"
                      onClick={() => handleSelect(c)}
                      className={`group w-full flex items-center gap-3 p-2.5 rounded-lg border-none bg-transparent cursor-pointer text-left transition-all duration-300 hover:translate-y-px
                        ${
                          isSelected
                            ? "bg-(--active-bg) shadow-(--active-shadow)"
                            : "hover:bg-(--active-bg-2)"
                        }`}
                      aria-label={`${c.name} ${c.code}`}
                    >
                      <span
                        className={`w-9 h-6 shrink-0 rounded-[4px] bg-cover bg-center sm:w-7 sm:h-[18px] ${c.flag}`}
                        aria-hidden="true"
                      />
                      <div className="flex justify-between items-center w-full">
                        <span className="text-[0.95rem] font-semibold text-(--text-color)">
                          {t(`countries.${c.name}`)}
                        </span>
                        <div className="w-[30%] min-w-[52px] min-h-[48px] flex items-center justify-center bg-(--input-bg) border border-(--input-border) rounded-lg text-(--muted-color) font-bold transition-all group-hover:border-(--primary-color)">
                          {c.code}
                        </div>
                      </div>
                    </Button>
                  );
                })}

                {filtered.length === 0 && (
                  <div className="py-8 text-center text-(--muted-color) text-sm font-black">
                    {t("common.no_results_found") || "No countries found."}
                  </div>
                )}
              </div>
            </div>
          </motion.div>
        )}
      </AnimatePresence>
    </Modal>
  );
};

export default CountryCodeModal;

```

## File: connectfy-client/src/components/Modal/SaveChangesModal/SaveChangesModal.tsx
```typescript
import { FC } from "react";
import { useTranslation } from "react-i18next";
import { TriangleAlert } from "lucide-react";
import Modal from "..";
import Button from "@/components/ui/CustomButton/Button/Button";

interface Props {
  open: boolean;
  handleSave: () => void;
  handleCancel: () => void;
  handleDiscardChanges: () => void;
  isLoading: boolean;
}

const SaveChangesModal: FC<Props> = ({
  open,
  handleSave,
  handleCancel,
  handleDiscardChanges,
  isLoading = false,
}) => {
  const { t } = useTranslation();

  const handleOverlayPointerDown = (e: React.MouseEvent<HTMLDivElement>) => {
    if (e.target === e.currentTarget) {
      handleCancel();
    }
  };

  return (
    <Modal
      open={open}
      onClose={handleCancel}
      onMouseDown={handleOverlayPointerDown}
    >
      {/* max-w-[560px] ilə modalın eni artırıldı */}
      <div className="bg-(--auth-main-bg) rounded-2xl p-8 max-w-[500px] w-[90%] shadow-(--card-shadow) animate-fade-in mx-auto">
        <div className="flex justify-center mb-4 animate-bounce-custom">
          <TriangleAlert size={50} className="text-(--error-color)" />
        </div>

        <h2 className="text-2xl font-bold text-(--text-primary) mb-3 text-center sm:text-xl">
          {t("common.unsaved_changes") || "Unsaved Changes"}
        </h2>

        <p className="text-[15px] text-(--text-secondary) mb-6 text-center leading-relaxed">
          {t("common.unsaved_changes_message") ||
            "You have unsaved changes. Do you want to save them before leaving?"}
        </p>

        {/* Mobildə alt-alta (flex-col), kiçik ekrandan etibarən yan-yana (sm:flex-row) */}
        <div className="flex flex-col sm:flex-row gap-3 justify-center">
          {/* Discard Button */}
          <Button
            className="w-full sm:flex-1 px-6 py-3 rounded-[10px] text-sm font-semibold cursor-pointer transition-all duration-200 bg-(--input-bg) text-(--text-primary) hover:bg-red-500/10 hover:text-red-500 dark:bg-(--auth-glass-bg) dark:hover:bg-red-500/20 disabled:opacity-50 disabled:cursor-not-allowed"
            onClick={handleDiscardChanges}
            disabled={isLoading}
            title={t("common.discard")}
          />

          {/* Cancel Button */}
          {/* <Button
            className="w-full sm:flex-1 px-6 py-3 rounded-[10px] text-sm font-semibold cursor-pointer transition-all duration-200 bg-(--input-bg) text-(--text-primary) hover:bg-(--input-border) dark:bg-(--auth-glass-bg) disabled:opacity-50 disabled:cursor-not-allowed"
            onClick={handleCancel}
            disabled={isLoading}
            title={t("common.cancel")}
          /> */}

          {/* Save Button */}
          <Button
            className="w-full sm:flex-1 px-6 py-3 rounded-[10px] text-sm font-semibold cursor-pointer transition-all duration-200 bg-linear-to-br from-(--third-color) to-(--hover-bg) text-white shadow-(--shadow-color) hover:shadow-(--active-shadow) flex items-center justify-center"
            onClick={handleSave}
            disabled={isLoading}
            isLoading={isLoading}
            title={t("common.save")}
          />
        </div>
      </div>
    </Modal>
  );
};

export default SaveChangesModal;

```

## File: connectfy-client/src/components/Modal/AuthenticateModal/AuthenticateModal.tsx
```typescript
import { FC, Fragment, useCallback, useEffect } from "react";
import { useTranslation } from "react-i18next";
import PasswordInput from "@/components/ui/CustomInput/PasswordInput/PasswordInput";
import { useFormik } from "formik";
import { checkEmptyString } from "@/common/utils/checkValues";
import { PROVIDER, TOKEN_TYPE } from "@/common/enums/enums";
import Modal from "..";
import { snack } from "@/common/utils/snackManager";
import { GoogleLogin } from "@react-oauth/google";
import { IAuthenticateUser } from "@/modules/auth/types/types";
import { useAuthenticateUserMutation } from "@/modules/auth/api/api";
import { useAuthStore } from "@/store/zustand/useAuthStore";
import { useErrors } from "@/hooks/useErrors";
import Button from "@/components/ui/CustomButton/Button/Button";
import { useUser } from "@/context/UserContext";

interface Props {
  open: boolean;
  onClose: () => void;
  onAuthenticate: () => void;
  authType: TOKEN_TYPE;
}

const AuthenticateModal: FC<Props> = ({
  open,
  onClose,
  onAuthenticate,
  authType,
}) => {
  const { setToken } = useAuthStore();
  const { t } = useTranslation();
  const { showResponseErrors } = useErrors();

  const [
    authenticateUser,
    { isLoading: LOADING_AUTHENTICATE_USER, error: ERROR_AUTHENTICATE_USER },
  ] = useAuthenticateUserMutation();

  const { user } = useUser();

  const { provider } = user ?? {};
  const usesPasswordAuth = provider === PROVIDER.PASSWORD;
  const usesOAuth =
    provider === PROVIDER.GOOGLE || provider === PROVIDER.FACEBOOK;

  const initialState: IAuthenticateUser = {
    password: null,
    type: authType,
    idToken: null,
  };

  const validate = ({
    password,
    idToken,
  }: IAuthenticateUser): Record<string, any> => {
    const errors: Record<string, any> = {};

    if (usesPasswordAuth && (!password || !checkEmptyString(password))) {
      errors.password = t("error_messages.this_field_required");
    }

    if (usesOAuth && (!idToken || !checkEmptyString(idToken)))
      errors.idToken = t("error_messages.this_field_required");

    return errors;
  };

  const formik = useFormik({
    initialValues: initialState,
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    validate: (values) => validate(values),
    onSubmit: async (values, { resetForm }) => {
      try {
        const res = await authenticateUser(values).unwrap();
        setToken({
          token: res.token,
          type: "authenticateToken",
        });
        onAuthenticate();
        resetForm();
        onClose();
      } catch (error) {
        showResponseErrors(error);
      }
    },
  });

  const handleOverlayPointerDown = (e: React.MouseEvent<HTMLDivElement>) => {
    if (LOADING_AUTHENTICATE_USER) return;
    if (e.target === e.currentTarget) {
      onClose();
    }
  };

  const handleSubmit = () => {
    if (usesPasswordAuth) {
      formik.submitForm();
    }
  };

  const isButtonsDisabled =
    LOADING_AUTHENTICATE_USER || ERROR_AUTHENTICATE_USER;

  const globalKeyDown = useCallback(
    (e: KeyboardEvent) => {
      if (!open) return;

      if (e.key === "Enter") {
        e.preventDefault();
        if (isButtonsDisabled || !formik.values.password) return;
        formik.submitForm();
      }

      if (e.key === "Escape") {
        e.preventDefault();
        if (isButtonsDisabled) return;
        onClose();
      }
    },
    [open, isButtonsDisabled, formik, onClose],
  );

  const handleGoogleSuccess = async (tokenResponse: any) => {
    try {
      const idToken = tokenResponse.credential;

      if (!idToken) {
        snack.error(t("error_messages.process_failed"));
        return;
      }

      const finalValues = {
        ...formik.values,
        idToken,
      };

      const res = await authenticateUser(finalValues).unwrap();
      setToken({
        token: res.token,
        type: "authenticateToken",
      });
      onAuthenticate();
      formik.resetForm();
      onClose();
    } catch (error) {
      showResponseErrors(error);
    }
  };

  useEffect(() => {
    if (!open) return;

    document.addEventListener("keydown", globalKeyDown);
    return () => {
      document.removeEventListener("keydown", globalKeyDown);
    };
  }, [open, globalKeyDown]);

  return (
    <Modal open={open} onClose={onClose} onMouseDown={handleOverlayPointerDown}>
      <div className="bg-(--auth-main-bg) rounded-2xl p-6 sm:p-8 max-w-[440px] w-[90%] shadow-(--card-shadow) animate-fade-in mx-auto">
        <h2 className="text-2xl font-bold text-(--text-primary) mb-3 text-center">
          {t("common.authentication_required")}
        </h2>

        <p className="text-[15px] text-(--text-secondary)] mb-6 text-center leading-relaxed">
          {usesPasswordAuth
            ? t("common.enter_password_to_continue")
            : t("common.verify_identity_to_continue")}
        </p>

        {usesPasswordAuth && (
          <Fragment>
            <div className="relative mb-2">
              <PasswordInput
                inputSize="large"
                title={t("common.password")}
                value={formik.values.password || ""}
                onChange={(e) => {
                  formik.setFieldValue("password", e.target.value || null);
                }}
                disabled={LOADING_AUTHENTICATE_USER}
                autoFocus
                isError={!!formik.errors.password}
              />
            </div>
          </Fragment>
        )}

        {usesPasswordAuth ? (
          <div className="flex gap-3 justify-center mt-6">
            <Button
              className="flex-1 min-w-[120px] h-11 flex items-center justify-center px-7 py-3 rounded-[10px] text-sm font-semibold transition-all duration-200 bg-(--input-bg) text-(--text-primary) hover:bg-(--input-border)"
              onClick={onClose}
              disabled={LOADING_AUTHENTICATE_USER}
              title={t("common.cancel")}
            />

            <Button
              className="flex-1 min-w-[120px] h-11 flex items-center justify-center px-7 py-3 rounded-[10px] text-sm font-semibold transition-all duration-600 bg-linear-to-br from-(--third-color) to-(--hover-bg) text-white shadow-(--shadow-color) hover:shadow-(--active-shadow)"
              onClick={handleSubmit}
              disabled={
                LOADING_AUTHENTICATE_USER ||
                (usesPasswordAuth && !formik.values.password)
              }
              isLoading={LOADING_AUTHENTICATE_USER}
              title={t("common.submit")}
            />
          </div>
        ) : (
          <div className="flex items-center justify-center gap-3 w-full mt-6">
            <Button
              className="w-full sm:flex-1 min-w-[120px] h-11 flex items-center justify-center px-7 py-3 rounded-[10px] text-sm font-semibold transition-all duration-200 bg-(--input-bg) text-(--text-primary) hover:bg-(--input-border)"
              onClick={onClose}
              disabled={LOADING_AUTHENTICATE_USER}
              title={t("common.cancel")}
            />

            <div className="relative w-full sm:flex-[1.3] min-w-0">
              <Button
                className="w-full h-11 flex items-center justify-center px-7 py-3 rounded-[10px] text-sm font-semibold transition-all duration-200 bg-linear-to-br from-(--third-color) to-(--hover-bg) text-white shadow-(--shadow-color) hover:shadow-(--active-shadow)"
                onClick={handleSubmit}
                disabled={LOADING_AUTHENTICATE_USER}
                title={t("common.authenticate")}
                isLoading={LOADING_AUTHENTICATE_USER}
              />

              <div className="absolute inset-0 opacity-0 z-10 overflow-hidden rounded-[10px] [&_div]:w-full! [&_div]:h-full! [&_div]:min-w-full! [&_div]:min-h-full! [&_iframe]:w-full! [&_iframe]:h-full! [&_iframe]:min-w-full! [&_iframe]:min-h-full! [&_iframe]:opacity-0">
                <GoogleLogin
                  onSuccess={handleGoogleSuccess}
                  onError={() =>
                    snack.error(t("error_messages.google_login_failed"))
                  }
                  useOneTap={false}
                  width="100%"
                  locale="en"
                  theme="filled_blue"
                  size="large"
                  type="standard"
                  shape="rectangular"
                  text="continue_with"
                />
              </div>
            </div>
          </div>
        )}
      </div>
    </Modal>
  );
};

export default AuthenticateModal;

```

## File: connectfy-client/src/components/Modal/DatePickerModal/DatePickerModal.tsx
```typescript
import { FC } from "react";
import Modal from "..";
import { useTranslation } from "react-i18next";
import Button from "@/components/ui/CustomButton/Button/Button";
import { ArrowLeft, ArrowRight } from "lucide-react";

interface IProps {
  open: boolean;
  onClose: () => void;
  onSelectDate: (date: string) => void;
  onClear: () => void;
  selectedDate: Date | null;
  currentMonth: Date;
  setCurrentMonth: (date: Date) => void;
  viewMode: "days" | "months" | "years";
  setViewMode: (mode: "days" | "months" | "years") => void;
  minDate: Date;
  maxDate: Date;
}

const DatePickerModal: FC<IProps> = ({
  open,
  onClose,
  onSelectDate,
  onClear,
  selectedDate,
  currentMonth,
  setCurrentMonth,
  viewMode,
  setViewMode,
  minDate,
  maxDate,
}) => {
  const { t } = useTranslation();
  const currentYear = new Date().getFullYear();

  const handleDateSelect = (date: Date) => {
    if (date >= minDate && date <= maxDate) {
      const year = date.getFullYear();
      const month = String(date.getMonth() + 1).padStart(2, "0");
      const day = String(date.getDate()).padStart(2, "0");
      onSelectDate(`${year}-${month}-${day}`);
      onClose();
      setViewMode("days");
    }
  };

  const handleMonthSelect = (month: number) => {
    const newDate = new Date(currentMonth);
    newDate.setMonth(month);
    setCurrentMonth(newDate);
    setViewMode("days");
  };

  const handleYearSelect = (year: number) => {
    const newDate = new Date(currentMonth);
    newDate.setFullYear(year);

    if (year >= minDate.getFullYear() && year <= maxDate.getFullYear()) {
      setCurrentMonth(newDate);
      setViewMode("months");
    }
  };

  const handleToday = () => {
    const today = new Date();
    const year = today.getFullYear();
    const month = String(today.getMonth() + 1).padStart(2, "0");
    const day = String(today.getDate()).padStart(2, "0");
    onSelectDate(`${year}-${month}-${day}`);
    setCurrentMonth(today);
    onClose();
    setViewMode("days");
  };

  const handleClearClick = () => {
    onClear();
    onClose();
    setViewMode("days");
  };

  const navigateMonth = (direction: "prev" | "next") => {
    const newDate = new Date(currentMonth);
    if (direction === "prev") {
      if (viewMode === "years") {
        const newYear = currentMonth.getFullYear() - 12;
        if (newYear >= minDate.getFullYear()) {
          newDate.setFullYear(newYear);
        }
      } else {
        newDate.setMonth(currentMonth.getMonth() - 1);
      }
    } else {
      if (viewMode === "years") {
        const newYear = currentMonth.getFullYear() + 12;
        if (newYear <= currentYear) {
          newDate.setFullYear(newYear);
        }
      } else {
        newDate.setMonth(currentMonth.getMonth() + 1);
      }
    }
    setCurrentMonth(newDate);
  };

  const getDaysInMonth = (date: Date) => {
    const year = date.getFullYear();
    const month = date.getMonth();
    const firstDay = new Date(year, month, 1);
    const lastDay = new Date(year, month + 1, 0);
    const days = [];

    for (let i = 0; i < firstDay.getDay(); i++) {
      days.push(null);
    }

    for (let i = 1; i <= lastDay.getDate(); i++) {
      days.push(new Date(year, month, i));
    }

    return days;
  };

  const getMonths = () => {
    const months = [];
    for (let i = 0; i < 12; i++) {
      months.push(i);
    }
    return months;
  };

  const getYears = () => {
    const currentYear = currentMonth.getFullYear();
    const startYear = Math.floor(currentYear / 12) * 12;
    const years = [];

    for (let i = startYear - 1; i < startYear + 14; i++) {
      years.push(i);
    }
    return years;
  };

  const isDateDisabled = (date: Date | null): boolean => {
    if (!date) return false;
    return date < minDate || date > maxDate;
  };

  const isMonthDisabled = (month: number): boolean => {
    const testDate = new Date(currentMonth.getFullYear(), month, 15);
    return testDate < minDate || testDate > maxDate;
  };

  const isYearDisabled = (year: number): boolean => {
    return year < minDate.getFullYear() || year > maxDate.getFullYear();
  };

  const days = getDaysInMonth(currentMonth);
  const months = getMonths();
  const years = getYears();
  const today = new Date();

  const weekDays = [
    t("calendar.days.sun"),
    t("calendar.days.mon"),
    t("calendar.days.tue"),
    t("calendar.days.wed"),
    t("calendar.days.thu"),
    t("calendar.days.fri"),
    t("calendar.days.sat"),
  ];

  const monthShortNames = [
    "jan",
    "feb",
    "mar",
    "apr",
    "may",
    "jun",
    "jul",
    "aug",
    "sep",
    "oct",
    "nov",
    "dec",
  ];

  const monthFullNames = [
    "january",
    "february",
    "march",
    "april",
    "may_full",
    "june",
    "july",
    "august",
    "september",
    "october",
    "november",
    "december",
  ];

  const renderHeader = () => {
    if (viewMode === "years") {
      const startYear = years[0];
      const endYear = years[years.length - 1];
      return `${startYear} - ${endYear}`;
    }

    if (viewMode === "months") {
      return currentMonth.getFullYear().toString();
    }

    const monthIndex = currentMonth.getMonth();
    const monthKey = monthFullNames[monthIndex];
    const monthName = t(`calendar.months.${monthKey}`);
    const year = currentMonth.getFullYear();

    return `${monthName} ${year}`;
  };

  const handleHeaderClick = () => {
    if (viewMode === "days") {
      setViewMode("months");
    } else if (viewMode === "months") {
      setViewMode("years");
    }
  };

  const isPrevDisabled = () => {
    if (viewMode === "years") {
      return years[0] <= minDate.getFullYear();
    }
    if (viewMode === "months") {
      return currentMonth.getFullYear() <= minDate.getFullYear();
    }
    const prevMonth = new Date(currentMonth);
    prevMonth.setMonth(currentMonth.getMonth() - 1);
    return prevMonth < minDate;
  };

  const isNextDisabled = () => {
    if (viewMode === "years") {
      return years[years.length - 1] >= maxDate.getFullYear();
    }
    if (viewMode === "months") {
      return currentMonth.getFullYear() >= maxDate.getFullYear();
    }
    const nextMonth = new Date(currentMonth);
    nextMonth.setMonth(currentMonth.getMonth() + 1);
    return nextMonth > maxDate;
  };

  return (
    <Modal open={open} onClose={onClose}>
      <div className="bg-(--card-bg) rounded-xl p-5 w-full max-w-[360px] mx-auto animate-in fade-in zoom-in-95 duration-200">
        {/* Header */}
        <div className="flex justify-between items-center mb-4">
          <Button
            className={`bg-transparent border-none text-2xl cursor-pointer py-2 px-3 rounded-lg transition-all text-(--text-color) font-light hover:bg-(--active-bg-2) hover:text-(--primary-color) ${
              isPrevDisabled() ? "opacity-30 cursor-not-allowed" : ""
            }`}
            onClick={() => !isPrevDisabled() && navigateMonth("prev")}
            disabled={isPrevDisabled()}
            type="button"
            icon={<ArrowLeft size={18} />}
          />
          <div
            className="font-semibold text-base text-(--text-color) cursor-pointer py-2 px-4 rounded-lg transition-all hover:bg-(--active-bg-2) hover:text-(--primary-color)"
            onClick={handleHeaderClick}
          >
            {renderHeader()}
          </div>
          <Button
            className={`bg-transparent border-none text-2xl cursor-pointer py-2 px-3 rounded-lg transition-all text-(--text-color) font-light hover:bg-(--active-bg-2) hover:text-(--primary-color) ${
              isNextDisabled() ? "opacity-30 cursor-not-allowed" : ""
            }`}
            onClick={() => !isNextDisabled() && navigateMonth("next")}
            disabled={isNextDisabled()}
            type="button"
            icon={<ArrowRight size={18} />}
          />
        </div>

        {/* Days View */}
        {viewMode === "days" && (
          <>
            <div className="grid grid-cols-7 gap-1 mb-3">
              {weekDays.map((day) => (
                <div
                  key={day}
                  className="text-center text-[13px] font-semibold text-(--muted-color) py-2 px-1"
                >
                  {day}
                </div>
              ))}
            </div>

            <div className="grid grid-cols-7 gap-1.5 mb-4">
              {days.map((date, index) => (
                <div
                  key={index}
                  className={`flex items-center justify-center h-11 rounded-lg cursor-pointer text-[15px] font-medium transition-all text-(--text-color)
                    ${!date ? "cursor-default" : ""}
                    ${date && !isDateDisabled(date) ? "hover:bg-(--active-bg-2) hover:text-(--primary-color)" : ""}
                    ${date && selectedDate && date.toDateString() === selectedDate.toDateString() ? "bg-(--primary-color) text-white!" : ""}
                    ${date && date.toDateString() === today.toDateString() ? "border-2 border-(--primary-color) text-(--primary-color) font-bold" : ""}
                    ${date && date.getMonth() !== currentMonth.getMonth() ? "text-(--muted-color) opacity-40" : ""}
                    ${date && isDateDisabled(date) ? "opacity-30 cursor-not-allowed" : ""}
                  `}
                  onClick={() =>
                    date && !isDateDisabled(date) && handleDateSelect(date)
                  }
                >
                  {date ? date.getDate() : ""}
                </div>
              ))}
            </div>
          </>
        )}

        {/* Months View */}
        {viewMode === "months" && (
          <div className="grid grid-cols-3 gap-2.5 mb-4">
            {months.map((month) => (
              <div
                key={month}
                className={`flex items-center justify-center h-12 rounded-xl cursor-pointer text-[15px] font-medium transition-all text-(--text-color)
                  ${month === currentMonth.getMonth() ? "bg-(--primary-color) text-white" : "hover:bg-(--active-bg-2) hover:text-(--primary-color)"}
                  ${isMonthDisabled(month) ? "opacity-30 cursor-not-allowed" : ""}
                `}
                onClick={() =>
                  !isMonthDisabled(month) && handleMonthSelect(month)
                }
              >
                {t(`calendar.months.${monthShortNames[month]}`)}
              </div>
            ))}
          </div>
        )}

        {/* Years View */}
        {viewMode === "years" && (
          <div className="grid grid-cols-3 gap-2.5 mb-4">
            {years.map((year) => (
              <div
                key={year}
                className={`flex items-center justify-center h-12 rounded-xl cursor-pointer text-[15px] font-medium transition-all text-(--text-color)
                  ${year === currentMonth.getFullYear() ? "bg-(--primary-color) text-white" : "hover:bg-(--active-bg-2) hover:text-(--primary-color)"}
                  ${isYearDisabled(year) ? "opacity-30 cursor-not-allowed" : ""}
                `}
                onClick={() => !isYearDisabled(year) && handleYearSelect(year)}
              >
                {year}
              </div>
            ))}
          </div>
        )}

        {/* Footer */}
        <div className="flex justify-between gap-3 mt-5 pt-4 border-t border-(--input-border)">
          <Button
            className="flex-1 bg-transparent border border-(--input-border) py-3 px-5 rounded-xl cursor-pointer text-[15px] font-semibold transition-all text-(--text-color) hover:bg-(--active-bg-2) hover:text-(--primary-color) hover:border-(--primary-color)"
            onClick={handleClearClick}
            type="button"
            title={t("common.clear")}
          />
          <Button
            className="flex-1 bg-(--primary-color) border border-(--primary-color) py-3 px-5 rounded-xl cursor-pointer text-[15px] font-semibold transition-all text-white hover:bg-(--hover-bg) hover:border-(--hover-bg)"
            onClick={handleToday}
            type="button"
            title={t("common.today")}
          />
        </div>
      </div>
    </Modal>
  );
};

export default DatePickerModal;

```

## File: connectfy-client/src/components/Modal/GlobalModals/GlobalModals.tsx
```typescript
import { Fragment } from "react";
import { useAvatarModalStore } from "@/store/zustand/useAvatarModalStore";
import ShowAvatarModal from "../AvatarModal/ShowAvatarModal/ShowAvatarModal";
import ChangeAvatarModal from "@/components/Modal/AvatarModal/ChangeAvatarModal/ChangeAvatartModal";
import UploadAvatarModal from "@/components/Modal/AvatarModal/UploadAvatarModal/UploadAvatarModal"; // Yolunu düzəlt
import { useGetAccountQuery } from "@/modules/profile/api/api";
import { useUser } from "@/context/UserContext";
import ChangeDefaultAvatarModal from "../AvatarModal/ChangeDefaultAvatarModal/ChangeDefaultAvatarModal";

const GlobalModals = () => {
  const store = useAvatarModalStore();
  const { user } = useUser();

  const { avatar, isProfileLoading } = useGetAccountQuery(undefined, {
    skip:
      !user?._id ||
      !!store.avatarObj ||
      (!store.isChangeModalOpen && !store.isUploadModalOpen),
    selectFromResult: (result) => ({
      avatar: result.data?.avatar,
      isProfileLoading: result.isLoading,
    }),
  });

  const currentAvatar = store.avatarObj || avatar;

  return (
    <Fragment>
      {/* 1. Show Avatar Modal */}
      <ShowAvatarModal
        open={store.isShowModalOpen}
        onClose={store.onCloseShowModal}
        avatarUrl={store.avatarUrl}
        username={store.username}
        userId={store.userId}
      />

      {/* 2. Change Avatar Modal */}
      <ChangeAvatarModal
        open={store.isChangeModalOpen}
        onClose={store.onCloseChangeModal}
        profileId={store.profileId}
        avatar={currentAvatar}
        isProfileLoading={isProfileLoading}
      />

      {/* 3. Upload Avatar Modal */}
      <UploadAvatarModal
        open={store.isUploadModalOpen}
        onClose={store.onCloseUploadModal}
        profileId={store.profileId}
        isProfileLoading={isProfileLoading}
      />

      {/* 4. Change Default Avatar Modal */}
      <ChangeDefaultAvatarModal
        open={store.isSetDefaultModalOpen}
        onClose={store.onCloseSetDefaultModal}
        profileId={store.profileId}
        defaultAvatar={store.defaultAvatarObj!}
      />
    </Fragment>
  );
};

export default GlobalModals;

```

## File: connectfy-client/src/components/Form/OTPForm/OTPForm.tsx
```typescript
import { ChangeEvent, useEffect, useRef, useState } from "react";
import Input from "@/components/ui/CustomInput/Input/Input";
import { useAppNavigation } from "@/hooks/useAppNavigation";

type OTPProps = {
  length: number;
  name: string;
  onComplete?: (code: string) => void;
  onChange: (value: string | null) => void;
  onKeyDown: (e: React.KeyboardEvent<HTMLElement>) => void;
};

export default function OTPForm({
  length,
  name,
  onComplete,
  onChange,
  onKeyDown,
}: OTPProps) {
  const { navigate } = useAppNavigation();
  const [values, setValues] = useState<string[]>(() => Array(length).fill(""));
  const inputsRef = useRef<(HTMLInputElement | null)[]>([]);
  const lastEmittedRef = useRef<string | null>(null);
  const lastCompleteRef = useRef<string | null>(null);

  const focus = (i: number) => inputsRef.current[i]?.focus();
  const select = (i: number) => inputsRef.current[i]?.select();

  useEffect(() => {
    const firstEmpty = inputsRef.current.findIndex((i) => !i || i.value === "");
    const idx = firstEmpty === -1 ? Math.max(0, length - 1) : firstEmpty;
    inputsRef.current[idx]?.focus();
  }, [length]);

  useEffect(() => {
    const code = values.join("");
    if (code === lastEmittedRef.current) return;
    lastEmittedRef.current = code;
    if (code === "") onChange(null);
    else onChange(code);
    if (code.length === length && lastCompleteRef.current !== code) {
      lastCompleteRef.current = code;
      onComplete?.(code);
    }
  }, [values, length, onChange, onComplete]);

  const handleChange = (e: ChangeEvent<HTMLInputElement>, idx: number) => {
    const raw = e.target.value;
    if (!/^\d*$/.test(raw)) return;
    const char = raw.slice(-1);
    setValues((prev) => {
      const next = [...prev];
      next[idx] = char;
      if (char !== "" && idx < length - 1) {
        setTimeout(() => focus(idx + 1), 0);
      }
      return next;
    });
  };

  const handleKeyDown = (
    e: React.KeyboardEvent<HTMLInputElement>,
    idx: number,
  ) => {
    const key = e.key;
    if (key === "ArrowLeft") {
      e.preventDefault();
      if (idx > 0) focus(idx - 1);
    } else if (key === "ArrowRight") {
      e.preventDefault();
      if (idx < length - 1) focus(idx + 1);
    } else if (key === "Backspace") {
      e.preventDefault();
      setValues((prev) => {
        const next = [...prev];
        next[idx] = "";
        return next;
      });
      if (idx > 0) {
        focus(idx - 1);
        select(idx - 1);
      }
    } else if (key === "Delete") {
      e.preventDefault();
      setValues((prev) => {
        const next = [...prev];
        next[idx] = "";
        return next;
      });
    } else if (key === "Enter") {
      e.preventDefault();
      const code = values.join("").trim();
      if (code.length === length) onKeyDown(e);
    } else if (key === "Escape") {
      e.preventDefault();
      navigate(-1);
    }
  };

  const handlePaste = (
    e: React.ClipboardEvent<HTMLInputElement>,
    idx: number,
  ) => {
    e.preventDefault();
    const text = e.clipboardData.getData("text").replace(/\D/g, "");
    if (!text) return;
    setValues((prev) => {
      const next = [...prev];
      let writeCount = 0;
      for (let i = 0; i < text.length && idx + i < length; i++) {
        next[idx + i] = text[i];
        writeCount++;
      }
      const lastIndex = Math.min(length - 1, idx + writeCount - 1);
      setTimeout(() => {
        inputsRef.current[lastIndex]?.focus();
        const joined = next.join("");
        if (joined.length === length) onComplete?.(joined);
      }, 0);
      return next;
    });
  };

  return (
    <div className="flex justify-between gap-2 md:gap-3" id="otp-container">
      {Array.from({ length }).map((_, i) => (
        <Input
          key={i}
          className="otp-input w-12 h-14 md:w-14 md:h-16 text-center text-2xl font-bold bg-slate-50 dark:bg-card-dark border border-slate-200 dark:border-emerald-900/30 rounded-xl focus:ring-primary focus:border-primary transition-all text-slate-900 dark:text-white"
          maxLength={1}
          type="text"
          value={values[i] ?? ""}
          onChange={(e) => handleChange(e, i)}
          onKeyDown={(e) => handleKeyDown(e, i)}
          onPaste={(e) => handlePaste(e, i)}
          inputMode="numeric"
          name={name}
          aria-label={`digit-${i + 1}`}
          autoComplete="off"
          ref={(el) => {
            inputsRef.current[i] = el;
          }}
        />
      ))}
    </div>
  );
}

```

## File: connectfy-client/src/components/Form/PhoneNumberForm/PhoneNumberForm.tsx
```typescript
import { FC, Fragment, useEffect, useMemo, useState } from "react";
import { COUNTRIES } from "@/common/constants/constants";
import { useTranslation } from "react-i18next";
import Input from "@/components/ui/CustomInput/Input/Input.tsx";
import useBoolean from "@/hooks/useBoolean";
import CountryCodeModal from "@/components/Modal/CountryCodeModal/CountryCodeModal";
import { IPhoneNumber } from "@/modules/auth/types/types";
import Button from "@/components/ui/CustomButton/Button/Button";
import { formatPhoneNumber } from "@/common/utils/formatValues";

interface Props {
  name: string;
  value?: IPhoneNumber | null;
  blur?: boolean;
  onBlur?: () => void;
  onChange: (value: IPhoneNumber | null) => void;
  onKeyDown?: (e: React.KeyboardEvent<HTMLElement>) => void;
}

const PhoneNumberForm: FC<Props> = ({
  name,
  value,
  onBlur,
  blur,
  onChange,
  onKeyDown,
}) => {
  const { t } = useTranslation();

  const [countryKey, setCountryKey] = useState<string>(COUNTRIES[0].key);
  const [hasLengthError, setHasLengthError] = useState<boolean>(false);

  const country = useMemo(
    () => COUNTRIES.find((c) => c.key === countryKey) || COUNTRIES[0],
    [countryKey],
  );

  const countryModal = useBoolean();

  const [fieldValue, setFieldValue] = useState<string | null>(null);

  const displayValue = useMemo(() => {
    return formatPhoneNumber(fieldValue || "", country.format || "");
  }, [fieldValue, country]);

  useEffect(() => {
    if (fieldValue)
      onChange({
        countryCode: country.code,
        number: fieldValue,
        fullPhoneNumber: country.code + fieldValue,
      });
    else onChange(null);
  }, [fieldValue, country.code]);

  useEffect(() => {
    if (fieldValue?.length !== country.numberLength) setHasLengthError(true);
    else setHasLengthError(false);
  }, [fieldValue, country.numberLength]);

  useEffect(() => {
    if (value) {
      const { countryCode, number } = value;

      const country = COUNTRIES.find((c) => c.code === countryCode);

      if (country) {
        setCountryKey(country.key);
        setFieldValue(number);
      }
    }
  }, [value]);

  return (
    <Fragment>
      <div id="phone-number-form" className="flex flex-col w-full">
        {/* Ana konteyner: CSS-dəki .phone-number-form */}
        <div className="flex w-full gap-3 items-start">
          {/* Ölkə kodu hissəsi: CSS-dəki .country-code (30%) */}

          <Button
            type="button"
            tooltip={t("common.select_country")}
            onClick={countryModal.onOpen}
            // Tailwind ilə bütün stillər bura keçirildi
            className={`
                  flex items-center justify-center gap-2
                  md:w-[130px] w-[105px] h-[58px] rounded-lg border transition-all duration-300
                  bg-(--input-bg) border-(--input-border)
                  hover:border-(--primary-color)
                `}
          >
            <span className={`${country.flag} scale-125`}></span>
            <span className="text-(--text-primary) font-medium">
              {country.code}
            </span>
          </Button>

          {/* Telefon nömrəsi hissəsi: CSS-dəki .phone-number (70%) */}
          <div className="w-full">
            <Input
              className="w-full px-5 py-4 h-[58px] rounded-xl text-(--text-primary) outline-none transition-all duration-200 placeholder:text-(--text-secondary)/50 focus:ring-2 focus:ring-[#34d399]/50"
              style={{
                backgroundColor: "var(--input-bg)",
                border: "1px solid var(--input-border)",
              }}
              title={t("common.phoneNumber")}
              name={name}
              value={displayValue}
              onChange={(e) => {
                const val = e.target.value;
                const numericValue = val.replace(/\D/g, "");

                if (!country || numericValue.length > country.numberLength)
                  return;

                setFieldValue(numericValue || null);
              }}
              onBlur={onBlur}
              inputMode="numeric"
              onKeyDown={(e) => (onKeyDown ? onKeyDown(e) : undefined)}
              isError={hasLengthError && blur}
              maxLength={country.totalLength}
            />
          </div>
        </div>

        {/* Xəta mesajı */}
        {hasLengthError && blur && (
          <h6 className="mt-2 text-red-500 text-sm font-medium">
            {t("error_messages.invalid_phone_number_length")}
          </h6>
        )}
      </div>

      <CountryCodeModal
        open={countryModal.open}
        onClose={countryModal.onClose}
        onSelect={(countryKey) => setCountryKey(countryKey.key)}
      />
    </Fragment>
  );
};

export default PhoneNumberForm;

```

## File: connectfy-client/src/components/Form/GenderForm/GenderForm.tsx
```typescript
import "./genderForm.style.css";
import { useTranslation } from "react-i18next";
import { GENDER } from "@/common/enums/enums";
import { FormikProps } from "formik";
import { IGoogleSignupForm, ISignupForm } from "@/modules/auth/types/types";
import { FC, useCallback } from "react";
import Input from "@/components/ui/CustomInput/Input/Input";

interface Props {
  formik: FormikProps<ISignupForm> | FormikProps<IGoogleSignupForm>;
  formId?: string;
}

const GenderForm: FC<Props> = ({ formik, formId = "default" }) => {
  const { t } = useTranslation();

  const changeGender = useCallback(
    (value: GENDER) => {
      if (formik.values.gender === value) formik.setFieldValue("gender", null);
      else formik.setFieldValue("gender", value);
    },
    [formik],
  );

  return (
    <div className="gender-group">
      <Input
        autoComplete="off"
        type="radio"
        id={`male-${formId}`}
        name={`gender-${formId}`}
        value={GENDER.MALE}
        checked={formik.values.gender === GENDER.MALE}
        onClick={() => changeGender(GENDER.MALE)}
        onChange={() => {}}
      />
      <label htmlFor={`male-${formId}`}>{t("enum.male")}</label>

      <Input
        autoComplete="off"
        type="radio"
        id={`female-${formId}`}
        name={`gender-${formId}`}
        value={GENDER.FEMALE}
        checked={formik.values.gender === GENDER.FEMALE}
        onClick={() => changeGender(GENDER.FEMALE)}
        onChange={() => {}}
      />
      <label htmlFor={`female-${formId}`}>{t("enum.female")}</label>

      <Input
        autoComplete="off"
        type="radio"
        id={`other-${formId}`}
        name={`gender-${formId}`}
        value={GENDER.OTHER}
        checked={formik.values.gender === GENDER.OTHER}
        onClick={() => changeGender(GENDER.OTHER)}
        onChange={() => {}}
      />
      <label htmlFor={`other-${formId}`}>{t("enum.other")}</label>
    </div>
  );
};

export default GenderForm;

```

## File: connectfy-client/src/components/Card/UserCard/UserCard.tsx
```typescript
import NoProfilePhotoIcon from "@/assets/icons/NoProfilePhotoIcon";
import { ROUTER } from "@/common/constants/routet";
import { useAppNavigation } from "@/hooks/useAppNavigation";
import { ISearchUserResult } from "@/modules/users/AllUsers/types/types";
import { FC } from "react";

interface IProps {
  user: ISearchUserResult;
  actionButtons?: React.ReactNode;
  onClick?: ({ userId }: { userId: string }) => void;
}

const UserCard: FC<IProps> = ({ user, actionButtons, onClick }) => {
  const { _id, fullName, username, avatar } = user;
  const { navigate } = useAppNavigation();

  return (
    <li
      className="flex items-center gap-3 px-3 py-2.5 rounded-xl transition-colors duration-150 cursor-pointer group"
      style={{ background: "transparent" }}
      onMouseEnter={(e) =>
        ((e.currentTarget as HTMLLIElement).style.background =
          "var(--active-bg-2)")
      }
      onMouseLeave={(e) =>
        ((e.currentTarget as HTMLLIElement).style.background = "transparent")
      }
    >
      {/* Avatar */}
      <div className="relative w-11 h-11 min-w-[44px] rounded-full overflow-hidden ring-2 ring-(--active-bg) shrink-0">
        {avatar?.url ? (
          <img
            src={avatar.url}
            alt={fullName}
            className="w-full h-full object-cover"
            loading="eager"
            fetchPriority="high"
            decoding="async"
          />
        ) : (
          <div className="flex items-center justify-center w-full h-full bg-(--active-bg-2)">
            <NoProfilePhotoIcon />
          </div>
        )}
      </div>

      {/* Info */}
      <div
        className="flex-1 min-w-0"
        onClick={() => {
          if (onClick) {
            onClick({ userId: _id });
          } else {
            navigate(`${ROUTER.USERS.PROFILE}/${_id}`);
          }
        }}
      >
        <p
          className="text-sm font-semibold leading-tight truncate"
          style={{ color: "var(--text-color)" }}
        >
          {fullName}
        </p>
        <p
          className="text-xs mt-0.5 truncate"
          style={{ color: "var(--muted-color)" }}
        >
          @{username}
        </p>
      </div>

      {actionButtons}
    </li>
  );
};

export default UserCard;

```

## File: connectfy-client/src/components/Card/SettingsCard/SettingCard.tsx
```typescript
import {
  CSSProperties,
  FC,
  ForwardRefExoticComponent,
  ReactNode,
  RefAttributes,
} from "react";
import "./settingCard.style.css";
import { LucideProps } from "lucide-react";

interface Props {
  header?: {
    icon?: ForwardRefExoticComponent<
      Omit<LucideProps, "ref"> & RefAttributes<SVGSVGElement>
    >;
    title?: string;
    subtitle?: string;
    iconStyle?: CSSProperties;
    headerStyle?: CSSProperties;
  };
  children?: ReactNode;
  contentStyle?: CSSProperties;
  cardStyle?: CSSProperties;
}

const SettingCard: FC<Props> = ({
  header,
  children,
  contentStyle,
  cardStyle,
}) => {
  return (
    <div className="setting-card" style={cardStyle}>
      <div className="setting-card-header">
        {header?.icon && (
          <div className="setting-icon" style={header.iconStyle}>
            <header.icon size={20} />
          </div>
        )}
        {(header?.title || header?.subtitle) && (
          <div className="setting-card-title">
            <h3>{header?.title}</h3>
            <p>{header?.subtitle}</p>
          </div>
        )}
      </div>
      <div style={contentStyle}>{children}</div>
    </div>
  );
};

export default SettingCard;

```

## File: connectfy-client/src/components/Card/ToggleCard/ToggleCard.tsx
```typescript
import {
  CSSProperties,
  FC,
  ForwardRefExoticComponent,
  RefAttributes,
} from "react";
import "./toggleCard.style.css";
import { LucideProps } from "lucide-react";
import ToggleSlider from "@/components/ui/CustomCheckbox/ToggleSlider/ToggleSlider";

interface Props {
  header?: {
    icon?: ForwardRefExoticComponent<
      Omit<LucideProps, "ref"> & RefAttributes<SVGSVGElement>
    >;
    title?: string;
    subtitle?: string;
    iconStyle?: CSSProperties;
    headerStyle?: CSSProperties;
  };
  slider: {
    checked: boolean;
    onClick: () => void;
  };
}

const ToggleCard: FC<Props> = ({ header, slider }) => {
  return (
    <div className="toggle-card">
      <div className="toggle-card-header" style={header?.headerStyle}>
        {header?.icon && (
          <div className="toggle-card-icon" style={header.iconStyle}>
            <header.icon size={20} />
          </div>
        )}
        {(header?.title || header?.subtitle) && (
          <div className="toggle-card-title">
            <h3>{header?.title}</h3>
            <p>{header?.subtitle}</p>
          </div>
        )}
        <ToggleSlider checked={slider.checked} onClick={slider.onClick} />
      </div>
    </div>
  );
};

export default ToggleCard;

```

## File: connectfy-client/src/components/Skeleton/Skeleton.tsx
```typescript
import React, { memo } from "react";
import "./skeleton.style.css";

export interface SkeletonProps {
  variant?: "text" | "rect" | "avatar" | "card" | "grid";
  width?: string | number;
  height?: string | number;
  rows?: number;
  className?: string;
  ariaLabel?: string;
}

export const Skeleton: React.FC<SkeletonProps> = memo(
  ({
    variant = "text",
    width,
    height,
    rows = 1,
    className = "",
    ariaLabel = "Loading content...",
  }) => {
    const style = {
      ...(width !== undefined && { width }),
      ...(height !== undefined && { height }),
    };

    const baseClass = `skeleton skeleton-${variant} ${className}`;

    if (variant === "text" && rows > 1) {
      return (
        <div
          role="status"
          aria-busy="true"
          className={`skeleton-text-wrapper ${className}`}
          style={style}
        >
          <span className="sr-only">{ariaLabel}</span>
          {Array.from({ length: rows }).map((_, i) => (
            <div
              key={i}
              className="skeleton skeleton-text"
              style={{ width: i === rows - 1 ? "70%" : "100%" }}
            />
          ))}
        </div>
      );
    }

    if (variant === "card") {
      return (
        <SkeletonCard
          width={width}
          className={className}
          ariaLabel={ariaLabel}
        />
      );
    }

    if (variant === "grid") {
      return (
        <div
          role="status"
          aria-busy="true"
          className={`skeleton-grid ${className}`}
          style={style}
        >
          <span className="sr-only">{ariaLabel}</span>
          {Array.from({ length: rows > 1 ? rows : 3 }).map((_, i) => (
            <div key={i} className="skeleton skeleton-rect" />
          ))}
        </div>
      );
    }

    return (
      <div role="status" aria-busy="true" className={baseClass} style={style}>
        <span className="sr-only">{ariaLabel}</span>
      </div>
    );
  },
);

Skeleton.displayName = "Skeleton";

interface SkeletonCardProps {
  width?: string | number;
  className?: string;
  ariaLabel?: string;
}

export const SkeletonCard: React.FC<SkeletonCardProps> = memo(
  ({ width, className = "", ariaLabel = "Loading card..." }) => {
    return (
      <div
        role="status"
        aria-busy="true"
        className={`skeleton-card-container ${className}`}
        style={width !== undefined ? { width } : undefined}
      >
        <span className="sr-only">{ariaLabel}</span>
        <div className="skeleton skeleton-rect skeleton-card-image" />
        <div className="skeleton-card-content">
          <div className="skeleton skeleton-text skeleton-card-title" />
          <div className="skeleton skeleton-text skeleton-card-desc" />
          <div
            className="skeleton skeleton-text skeleton-card-desc"
            style={{ width: "80%" }}
          />
        </div>
      </div>
    );
  },
);

SkeletonCard.displayName = "SkeletonCard";

```

## File: connectfy-client/src/components/Skeleton/profile/SocialLinkSkeleton.tsx
```typescript
import { memo } from "react";
import { Skeleton } from "../Skeleton";

const SocialLinkSkeleton = memo(() => {
  return (
    <section
      className="p-8 mb-8 transition-all duration-300 bg-(--auth-main-bg) rounded-[20px] border border-(--auth-glass-border) shadow-(--card-shadow)"
      role="status"
      aria-busy="true"
      aria-label="Loading bio..."
    >
      {/* Header */}
      <div className="flex flex-row items-center justify-between gap-3 mb-6 md:mb-8">
        <div className="flex items-center gap-2 md:gap-3">
          <Skeleton variant="text" width={160} height={28} />
        </div>
        <Skeleton variant="text" width={35} height={35} />
      </div>

      {/* Bio Content */}
      <div className="p-6 rounded-xl bg-(--info-card-bg) border border-(--info-card-border)">
        <Skeleton variant="text" width="100%" height="1em" className="mb-3" />
        <Skeleton variant="text" width="90%" height="1em" className="mb-3" />
        <Skeleton variant="text" width="75%" height="1em" />
      </div>
    </section>
  );
});

SocialLinkSkeleton.displayName = "SocialLinkSkeleton";

export default SocialLinkSkeleton;

```

## File: connectfy-client/src/components/Skeleton/profile/MainCardSkeleton.tsx
```typescript
import { memo } from "react";
import { Skeleton } from "../Skeleton";

const MainCardSkeleton = memo(() => {
  return (
    <section
      className="flex flex-col items-center gap-6 p-8 mb-8 transition-all bg-(--auth-main-bg) rounded-[24px] shadow-(--card-shadow) md:p-10"
      role="status"
      aria-busy="true"
      aria-label="Loading profile card..."
    >
      {/* Avatar */}
      <div className="relative shrink-0">
        <Skeleton
          variant="avatar"
          className="w-[140px]! h-[140px]! md:w-[140px]! sm:w-[120px]! xs:w-[100px]! border-4 border-(--primary-color)"
        />
      </div>

      {/* Identity */}
      <div className="flex flex-col items-center gap-2 w-full max-w-[220px]">
        <Skeleton variant="text" width="75%" height="1.75rem" />
        <Skeleton variant="text" width="50%" height="1rem" />
        <Skeleton variant="text" width="60%" height="0.875rem" />
      </div>

      {/* Stats Container */}
      <div className="flex flex-col items-center w-full max-w-[400px] gap-4 p-4 rounded-2xl bg-(--active-bg-2) sm:flex-row sm:gap-6 sm:p-6 md:px-8">
        {/* Friends Stat */}
        <div className="flex flex-row items-center justify-center flex-1 gap-3 sm:flex-col sm:gap-1.5">
          <Skeleton variant="text" width="50px" height="1.5rem" />
          <Skeleton variant="text" width="70px" height="0.8rem" />
        </div>

        {/* Divider */}
        <div
          className="w-full h-px opacity-30 bg-(--border-color) sm:w-px sm:h-10"
          aria-hidden="true"
        />

        {/* Blocked Stat */}
        <div className="flex flex-row items-center justify-center flex-1 gap-3 sm:flex-col sm:gap-1.5">
          <Skeleton variant="text" width="50px" height="1.5rem" />
          <Skeleton variant="text" width="70px" height="0.8rem" />
        </div>
      </div>
    </section>
  );
});

MainCardSkeleton.displayName = "MainCardSkeleton";

export default MainCardSkeleton;

```

## File: connectfy-client/src/components/Skeleton/profile/BioSkeleton.tsx
```typescript
import { memo } from "react";
import { Skeleton } from "../Skeleton";

const BioSkeleton = memo(() => {
  return (
    <section
      className="p-8 mb-8 transition-all duration-300 bg-(--auth-main-bg) rounded-[20px] border border-(--auth-glass-border) shadow-(--card-shadow)"
      role="status"
      aria-busy="true"
      aria-label="Loading bio..."
    >
      {/* Header */}
      <div className="flex flex-row items-center justify-between gap-3 mb-6 md:mb-8">
        <div className="flex items-center gap-2 md:gap-3">
          <Skeleton variant="text" width={160} height={28} />
        </div>
        <Skeleton variant="text" width={35} height={35} />
      </div>

      {/* Bio Content */}
      <div className="p-6 rounded-xl bg-(--info-card-bg) border border-(--info-card-border)">
        <Skeleton variant="text" width="100%" height="1em" className="mb-3" />
        <Skeleton variant="text" width="90%" height="1em" className="mb-3" />
        <Skeleton variant="text" width="75%" height="1em" />
      </div>
    </section>
  );
});

BioSkeleton.displayName = "BioSkeleton";

export default BioSkeleton;

```

## File: connectfy-client/src/components/Skeleton/profile/ProfileHeaderSkeleton.tsx
```typescript
import { memo } from "react";
import { Skeleton } from "../Skeleton";
import "@/modules/profile/ui/components/ProfileHeader/profileHeader.style.css";

const ProfileHeaderSkeleton = memo(() => {
  return (
    <header
      className="profile-header-actions"
      role="status"
      aria-busy="true"
      aria-label="Loading profile header..."
    >
      <div className="profile-header-left">
        <div className="profile-back-btn p-0 border-none bg-transparent hover:transform-none hover:bg-transparent cursor-default">
          <Skeleton variant="text" width={35} height={35} />
        </div>
        <div className="profile-page-title inline-block">
          <Skeleton variant="text" width={120} height={28} />
        </div>
      </div>

      <div className="profile-header-right">
        {Array.from({ length: 3 }).map((_, i) => (
          <div
            key={i}
            className="profile-icon-btn p-0 border-none bg-transparent hover:transform-none hover:bg-transparent cursor-default shadow-none"
          >
            <Skeleton variant="text" width={35} height={35} />
          </div>
        ))}
      </div>
    </header>
  );
});

ProfileHeaderSkeleton.displayName = "ProfileHeaderSkeleton";

export default ProfileHeaderSkeleton;

```

## File: connectfy-client/src/components/Skeleton/profile/PersonalInformationSkeleton.tsx
```typescript
import { memo } from "react";
import { Skeleton } from "../Skeleton";

const PersonalInformationSkeleton = memo(() => {
  const items = Array.from({ length: 5 });

  return (
    <section
      className="p-8 mb-8 px-4 md:px-8 transition-all duration-300 bg-(--auth-main-bg) rounded-[20px] border border-(--auth-glass-border) shadow-(--card-shadow)"
      role="status"
      aria-busy="true"
      aria-label="Loading personal information..."
    >
      {/* Header */}
      <div className="flex flex-row items-center justify-between gap-3 mb-6 md:mb-8">
        <div className="flex items-center gap-2 md:gap-3">
          <Skeleton variant="text" width={160} height={28} />
        </div>
        <Skeleton variant="text" width={35} height={35} />
      </div>

      {/* Grid */}
      <div className="grid grid-cols-1 gap-4 sm:grid-cols-2">
        {items.map((_, i) => (
          <div
            key={i}
            className="flex items-center gap-4 p-4 lg:px-5 bg-(--info-card-bg) border border-(--info-card-border) rounded-xl"
          >
            {/* Icon Box */}
            <Skeleton
              variant="rect"
              className="rounded-xl w-10 h-10 md:w-12 md:h-12 shrink-0"
            />

            {/* Content */}
            <div className="flex flex-col flex-1 gap-2 overflow-hidden">
              <Skeleton variant="text" width="40%" height="0.7rem" />
              <Skeleton variant="text" width="70%" height="1rem" />
            </div>
          </div>
        ))}
      </div>
    </section>
  );
});

PersonalInformationSkeleton.displayName = "PersonalInformationSkeleton";

export default PersonalInformationSkeleton;

```

## File: connectfy-client/src/components/Skeleton/notification/NotificationSkeleton.tsx
```typescript
const NotificationSkeleton = () => (
  <div
    className="transition-all duration-300 bg-(--auth-main-bg) border border-(--auth-glass-border) shadow-(--card-shadow) rounded-xl p-4 mb-3 mt-3"
    role="status"
    aria-busy="true"
    aria-label="Loading notifications..."
  >
    <div className="flex items-start gap-3">
      <div className="shrink-0 w-10 h-10 rounded-full bg-(--skeleton-base)" />
      <div className="flex-1 space-y-2">
        <div className="flex items-start justify-between gap-4">
          <div className="h-3.5 bg-(--skeleton-base) rounded-full w-2/3" />
          <div className="h-3 bg-(--skeleton-base) rounded-full w-10 shrink-0" />
        </div>
        <div className="h-3 bg-(--skeleton-base) rounded-full w-full" />
        <div className="h-3 bg-(--skeleton-base) rounded-full w-3/4" />
      </div>
    </div>
  </div>
);

export default NotificationSkeleton;

```

## File: connectfy-client/src/components/Skeleton/settings/SettingsSkeleton.tsx
```typescript
import "./settingsSkeleton.style.css";
import { Skeleton } from "../Skeleton";
import { FC } from "react";

interface SettingsSkeletonProps {
  count?: number;
}

export const SettingsSkeleton: FC<SettingsSkeletonProps> = ({ count = 1 }) => {
  return (
    <div className="settings-skeleton-container">
      {Array.from({ length: count }).map((_, index) => (
        <div className="settings-skeleton-wrapper" key={index}>
          {/* Header: icon + title/subtitle */}
          <div className="settings-skeleton-header">
            <Skeleton variant="avatar" width={40} height={40} ariaLabel="" />
            <div className="settings-skeleton-titles">
              <Skeleton variant="text" width={80} height={14} ariaLabel="" />
              <Skeleton variant="text" width={48} height={12} ariaLabel="" />
            </div>
          </div>

          {/* Children rows */}
          <div
            className="settings-skeleton-children"
            role="status"
            aria-busy="true"
            aria-label="Loading settings..."
          >
            <Skeleton variant="rect" width="100%" height={48} ariaLabel="" />
          </div>
        </div>
      ))}
    </div>
  );
};

```

## File: connectfy-client/src/components/ui/CustomCheckbox/Checkbox/Checkbox.tsx
```typescript
import React, { ChangeEvent, FC } from "react";

interface Props {
  id?: string;
  children?: React.ReactNode;
  checked: boolean;
  onChange: (e: ChangeEvent<HTMLInputElement>) => void;
  disabled?: boolean;
  className?: string;
}

const Checkbox: FC<Props> = ({
  id,
  children,
  checked,
  onChange,
  disabled = false,
  className,
}) => {
  const checkboxId = id ?? `checkbox-${Math.random()}`;

  return (
    <div className="flex items-center gap-3">
      <div className="relative">
        <input
          id={checkboxId}
          type="checkbox"
          checked={checked}
          onChange={onChange}
          disabled={disabled}
          className="peer sr-only"
        />
        <label
          htmlFor={checkboxId}
          className={`
            flex items-center justify-center
            w-5 h-5 rounded-md
            border-2 transition-all duration-200 cursor-pointer
            ${disabled ? "cursor-not-allowed opacity-50" : ""}
            ${
              checked
                ? "bg-(--primary-color) border-(--primary-color) shadow-checkbox-active"
                : "bg-white dark:bg-input-bg border-input-border hover:border-(--primary-color)/50"
            }
            ${className}
          `}
        >
          <svg
            className={`
              w-3.5 h-3.5 text-white transition-all duration-200
              ${checked ? "scale-100 opacity-100" : "scale-0 opacity-0"}
            `}
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
            strokeWidth="3.5"
          >
            <path
              strokeLinecap="round"
              strokeLinejoin="round"
              d="M5 13l4 4L19 7"
            />
          </svg>
        </label>
      </div>

      {children && (
        <label
          htmlFor={checkboxId}
          className={`
            text-sm font-medium select-none cursor-pointer
            text-text-color dark:text-text-color transition-colors
            ${disabled ? "cursor-not-allowed opacity-50" : "hover:text-(--primary-color)"}
          `}
        >
          {children}
        </label>
      )}
    </div>
  );
};

export default Checkbox;

```

## File: connectfy-client/src/components/ui/CustomCheckbox/ToggleSlider/ToggleSlider.tsx
```typescript
import { FC } from "react";
import "./toggleSlider.style.css";

interface Props {
  checked: boolean;
  onClick: () => void;
}

const ToggleSlider: FC<Props> = ({ checked, onClick }) => {
  return (
    <div
      className={`toggle-switch ${checked ? "active" : ""}`}
      onClick={onClick}
    >
      <div className="toggle-slider"></div>
    </div>
  );
};

export default ToggleSlider;

```

## File: connectfy-client/src/components/ui/Select/Dropdown/Dropdown.tsx
```typescript
import { FC, useRef, useEffect, useState, useCallback, ReactNode } from "react";
import { Check, ChevronRight, ChevronLeft } from "lucide-react";
import {
  motion,
  AnimatePresence,
  TargetAndTransition,
  VariantLabels,
  Transition,
} from "framer-motion";
import { useTranslation } from "react-i18next";
import Button from "../../CustomButton/Button/Button";
import TextTooltip from "@/components/Tooltip/TextTooltip";

export interface DropdownOption {
  label: string;
  value: string;
  icon?: React.ReactNode;
  onClick?: () => void;
  subMenu?: DropdownOption[];
  className?: string;
  isActive?: boolean; // YENİ: Alt menyuda seçilmiş elementi tanımaq üçün
}

interface DropdownProps {
  title?: string;
  options: DropdownOption[];
  selected?: string | null;
  icon?:
    | React.ReactNode
    | {
        open: React.ReactNode;
        close: React.ReactNode;
      };
  text?: string;
  animation?: {
    whileTap?: TargetAndTransition | VariantLabels;
    whileHover?: TargetAndTransition | VariantLabels;
    animate?: TargetAndTransition | VariantLabels;
  };
  tooltip?: string;
  className?: string;
  buttonClassName?: string;
  openAnimation?: {
    initial?: TargetAndTransition | VariantLabels;
    animate?: TargetAndTransition | VariantLabels;
    exit?: TargetAndTransition | VariantLabels;
    transition?: Transition;
  };
}

const Dropdown: FC<DropdownProps> = ({
  title,
  options,
  selected,
  animation,
  tooltip,
  icon,
  text,
  className,
  buttonClassName,
  openAnimation,
}) => {
  const { t } = useTranslation();
  const [open, setOpen] = useState(false);
  const [activeSubMenu, setActiveSubMenu] = useState<DropdownOption[] | null>(
    null,
  );
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const handleClickOutside = (e: MouseEvent) => {
      if (ref.current && !ref.current.contains(e.target as Node)) {
        setOpen(false);
        setTimeout(() => setActiveSubMenu(null), 200);
      }
    };
    document.addEventListener("mousedown", handleClickOutside);
    return () => document.removeEventListener("mousedown", handleClickOutside);
  }, []);

  const handleSelect = (opt: DropdownOption) => {
    if (opt.subMenu) {
      setActiveSubMenu(opt.subMenu);
    } else {
      opt.onClick?.();
      setOpen(false);
      setTimeout(() => setActiveSubMenu(null), 200);
    }
  };

  const renderButton = useCallback(() => {
    return (
      <motion.button
        type="button"
        onClick={() => setOpen((v) => !v)}
        whileTap={animation?.whileTap || { scale: 0.92 }}
        whileHover={
          animation?.whileHover || {
            backgroundColor: "rgba(128, 128, 128, 0.1)",
          }
        }
        className={`flex items-center justify-center w-9 h-9 rounded-lg border-none cursor-pointer bg-transparent text-(--text-color) ${buttonClassName}`}
      >
        <motion.span
          animate={open && animation?.animate ? animation.animate : {}}
          transition={{ duration: 0.28 }}
          className="flex items-center"
        >
          {typeof icon === "object" && icon !== null && "open" in icon
            ? open
              ? icon.open
              : icon.close
            : (icon as ReactNode)}
        </motion.span>
        {text}
      </motion.button>
    );
  }, [open, animation, icon, text, buttonClassName]);

  return (
    <div ref={ref} className={`relative ${className}`}>
      {tooltip ? (
        <TextTooltip position="top" text={tooltip}>
          {renderButton()}
        </TextTooltip>
      ) : (
        renderButton()
      )}

      <AnimatePresence>
        {open && (
          <motion.div
            initial={
              openAnimation?.initial || { opacity: 0, scale: 0.95, y: 8 }
            }
            animate={openAnimation?.animate || { opacity: 1, scale: 1, y: 0 }}
            exit={openAnimation?.exit || { opacity: 0, scale: 0.95, y: 8 }}
            className="absolute right-0 top-full mt-2 z-50 min-w-[190px] p-1.5 bg-(--auth-main-bg) border border-(--auth-glass-border) rounded-2xl shadow-xl overflow-hidden origin-top"
          >
            <AnimatePresence mode="wait">
              <motion.div
                key={activeSubMenu ? "sub" : "main"}
                initial={{ x: activeSubMenu ? 20 : -20, opacity: 0 }}
                animate={{ x: 0, opacity: 1 }}
                exit={{ x: activeSubMenu ? -20 : 20, opacity: 0 }}
                transition={openAnimation?.transition || { duration: 0.15 }}
              >
                {activeSubMenu ? (
                  <Button
                    onClick={() => setActiveSubMenu(null)}
                    className="flex items-center gap-2 w-full px-3 py-2 mb-1 text-xs font-bold text-(--muted-color) hover:text-(--text-color) border-b border-(--auth-glass-border) transition-colors bg-transparent border-x-0 border-t-0"
                  >
                    <ChevronLeft size={14} /> {t("common.back")}
                  </Button>
                ) : (
                  title && (
                    <div className="px-3 py-2 text-[10px] uppercase font-bold text-(--muted-color) opacity-60">
                      {title}
                    </div>
                  )
                )}

                {(activeSubMenu || options).map((opt) => {
                  // DƏYİŞİKLİK BURADADIR: Aktivlik həm 'selected' propundan, həm də option-ın özündən gələ bilər
                  const isActive = selected === opt.value || opt.isActive;

                  return (
                    <Button
                      key={opt.value}
                      onClick={() => handleSelect(opt)}
                      className={`flex items-center justify-between w-full px-3 py-2.5 rounded-xl text-sm transition-all border-none cursor-pointer bg-transparent group ${
                        isActive
                          ? "bg-(--primary-color)/10 text-(--primary-color) font-semibold"
                          : "text-(--text-color) hover:bg-black/5 dark:hover:bg-white/5"
                      } ${opt.className || ""}`}
                    >
                      <div className="flex items-center gap-2.5 font-semibold">
                        {opt.icon && <span>{opt.icon}</span>}
                        {opt.label}
                      </div>
                      {opt.subMenu ? (
                        <ChevronRight size={14} className="opacity-40" />
                      ) : (
                        isActive && <Check size={14} strokeWidth={3} />
                      )}
                    </Button>
                  );
                })}
              </motion.div>
            </AnimatePresence>
          </motion.div>
        )}
      </AnimatePresence>
    </div>
  );
};

export default Dropdown;

```

## File: connectfy-client/src/components/ui/Select/CustomSelect/CustomSelect.tsx
```typescript
import { FC, ForwardRefExoticComponent, Fragment, RefAttributes } from "react";
import "./customSelect.style.css";
import { ChevronRight, LucideProps } from "lucide-react";
import useBoolean from "@/hooks/useBoolean";
import SelectionModal from "@/components/Modal/SelectionModal/SelectionModal";

interface Selections {
  name: string;
  title?: string;
  onClick: () => void;
  icon: ForwardRefExoticComponent<
    Omit<LucideProps, "ref"> & RefAttributes<SVGSVGElement>
  >;
  key: string;
}

interface OptionProps {
  title: string;
  selections: Selections[];
  activeKey: string;
}

interface Props {
  buttonTitle: string;
  options: OptionProps;
}

const CustomSelect: FC<Props> = ({ buttonTitle, options }) => {
  const { open, onOpen, onClose } = useBoolean();
  const buttonName =
    options.selections.find((s) => s.key === options.activeKey)?.title ||
    buttonTitle;

  return (
    <Fragment>
      <div className="select-wrapper">
        <div className="custom-select" onClick={onOpen}>
          {buttonName}
        </div>

        <ChevronRight size={20} className="select-icon" />
      </div>

      {open && (
        <SelectionModal
          open={open}
          title={options.title}
          selections={options.selections}
          onClose={onClose}
          activeKey={options.activeKey}
        />
      )}
    </Fragment>
  );
};

export default CustomSelect;

```

## File: connectfy-client/src/components/ui/Select/NativeSelect/NativeSelect.tsx
```typescript
import { FC, useState, useRef, useEffect, useId, ReactNode } from "react";
import { ChevronDown, Check } from "lucide-react";
import { motion, AnimatePresence } from "framer-motion";
import Button from "../../CustomButton/Button/Button";

export interface ISelectOption {
  label: string | ReactNode;
  value: string | number;
}

interface ICustomSelectProps {
  options: ISelectOption[];
  value: string | number;
  onChange: (value: string | number) => void;
  placeholder?: string;
  title?: string;
  isFloating?: boolean;
  inputSize?: "small" | "medium" | "large" | "xlarge";
  icon?: ReactNode;
  disabled?: boolean;
  className?: string;
  visibleOptions?: number;
}

const OPTION_HEIGHT = 40; // py-2.5 (20px) + text-sm line-height (20px)
const LIST_PADDING = 16; // inner div py-2 (8px top + 8px bottom)

const NativeSelect: FC<ICustomSelectProps> = ({
  options,
  value,
  onChange,
  placeholder,
  title,
  isFloating = true,
  inputSize = "large",
  icon,
  disabled = false,
  className = "",
  visibleOptions = 5,
}) => {
  const [isOpen, setIsOpen] = useState(false);
  const containerRef = useRef<HTMLDivElement>(null);
  const selectId = useId();

  const maxHeight = OPTION_HEIGHT * visibleOptions + LIST_PADDING;

  const selectedOption = options.find((opt) => opt.value === value);
  const isLabelFloating =
    isFloating && (isOpen || (value !== undefined && value !== ""));

  const sizeConfig = {
    small: {
      padding: "py-[0.625rem] px-[0.75rem]",
      fontSize: "text-sm",
      labelLeft: icon ? "left-10" : "left-3",
      iconLeft: "left-2.5",
      floatingOffset: "-top-2",
    },
    medium: {
      padding: "py-[0.875rem] px-[1rem]",
      fontSize: "text-base",
      labelLeft: icon ? "left-11" : "left-4",
      iconLeft: "left-3",
      floatingOffset: "-top-2.5",
    },
    large: {
      padding: "py-[1rem] px-[1.25rem]",
      fontSize: "text-[1.0625rem]",
      labelLeft: icon ? "left-12" : "left-5",
      iconLeft: "left-4",
      floatingOffset: "-top-2",
    },
    xlarge: {
      padding: "py-[1.125rem] px-[1.5rem]",
      fontSize: "text-lg",
      labelLeft: icon ? "left-14" : "left-6",
      iconLeft: "left-4.5",
      floatingOffset: "-top-3.5",
    },
  };

  const currentSize = sizeConfig[inputSize];

  useEffect(() => {
    const handleClickOutside = (event: MouseEvent) => {
      if (
        containerRef.current &&
        !containerRef.current.contains(event.target as Node)
      ) {
        setIsOpen(false);
      }
    };
    document.addEventListener("mousedown", handleClickOutside);
    return () => document.removeEventListener("mousedown", handleClickOutside);
  }, []);

  return (
    <div
      className={`relative w-full min-w-[200px] ${className}`}
      ref={containerRef}
    >
      {/* Floating Label */}
      {title && (
        <label
          htmlFor={selectId}
          className={`
            absolute transition-all duration-300 pointer-events-none z-10
            ${
              isLabelFloating
                ? `${currentSize.floatingOffset} left-4 text-[13px] px-1 bg-(--input-bg)`
                : `top-1/2 -translate-y-1/2 ${currentSize.labelLeft} ${currentSize.fontSize} text-(--muted-color)`
            }
            ${disabled ? "opacity-50" : ""}
          `}
        >
          {title}
        </label>
      )}

      {/* İkon */}
      {icon && (
        <div
          className={`absolute ${currentSize.iconLeft} top-1/2 -translate-y-1/2 z-10 transition-colors duration-300 
          ${isOpen ? "text-(--primary-color)" : "text-(--muted-color)"}
          ${disabled ? "opacity-50" : ""}
        `}
        >
          {icon}
        </div>
      )}

      {/* Trigger Button */}
      <Button
        id={selectId}
        type="button"
        disabled={disabled}
        onClick={() => !disabled && setIsOpen(!isOpen)}
        className={`
          w-full flex items-center justify-between ${currentSize.padding} ${currentSize.fontSize}
          rounded-xl border transition-all duration-300 outline-none
          bg-(--input-bg) text-(--text-primary)
          ${icon ? "pl-12" : ""} 
          ${
            disabled
              ? "cursor-not-allowed opacity-60 bg-gray-100/10 border-gray-500/20"
              : "cursor-pointer border-(--auth-glass-border)"
          }
          ${isOpen ? "border-(--primary-color) ring-2 ring-(--primary-color)/10" : ""}
        `}
      >
        <span className="truncate">
          {selectedOption
            ? selectedOption.label
            : !isLabelFloating
              ? ""
              : placeholder}
        </span>
        <ChevronDown
          size={18}
          className={`transition-transform duration-300 ${isOpen ? "rotate-180 text-(--primary-color)" : ""} ${disabled ? "opacity-50" : ""}`}
        />
      </Button>

      {/* Options List */}
      <AnimatePresence>
        {isOpen && !disabled && (
          <motion.ul
            initial={{ opacity: 0, y: -10, scale: 0.95 }}
            animate={{ opacity: 1, y: 5, scale: 1 }}
            exit={{ opacity: 0, y: -10, scale: 0.95 }}
            transition={{ duration: 0.2, ease: "easeOut" }}
            style={{
              maxHeight: `${maxHeight}px`,
              scrollbarWidth: "thin",
              scrollbarColor: "var(--primary-color) transparent",
            }}
            className="
              absolute z-50 w-full mt-1 overflow-y-auto
              bg-(--auth-main-bg) border border-(--auth-glass-border)
              rounded-xl shadow-(--card-shadow)
              scrollbar-thin scrollbar-thumb-(--primary-color)/40 scrollbar-track-transparent
            "
          >
            <div className="py-2">
              {options.map((option) => {
                const isActive = option.value === value;
                return (
                  <li
                    key={option.value}
                    onClick={() => {
                      onChange(option.value);
                      setIsOpen(false);
                    }}
                    className={`
                      flex items-center justify-between px-4 py-2.5 text-sm font-semibold cursor-pointer transition-colors
                      ${
                        isActive
                          ? "bg-(--primary-color) text-white"
                          : "text-(--text-primary) hover:bg-(--primary-color)/10"
                      }
                    `}
                  >
                    <span className="truncate">{option.label}</span>
                    {isActive && (
                      <Check size={14} className="text-white shrink-0 ml-2" />
                    )}
                  </li>
                );
              })}
            </div>
          </motion.ul>
        )}
      </AnimatePresence>
    </div>
  );
};

export default NativeSelect;

```

## File: connectfy-client/src/components/ui/CustomInput/PasswordInput/PasswordInput.tsx
```typescript
import "./passwordInput.style.css";
import Input, {
  CustomInputProps,
} from "@/components/ui/CustomInput/Input/Input.tsx";
import React, { FC, useState } from "react";
import { useTranslation } from "react-i18next";
import { snack } from "@/common/utils/snackManager.ts";
import { Eye, EyeClosed, KeyRound, LockKeyhole } from "lucide-react";
import Button from "../../CustomButton/Button/Button";
import TextTooltip from "@/components/Tooltip/TextTooltip";

interface Props extends CustomInputProps {
  showGenerateButton?: boolean;
  onGenerate?: (value: string) => void;
}

const PasswordInput: FC<Props> = ({
  showGenerateButton,
  onGenerate,
  ...props
}) => {
  const { t } = useTranslation();
  const [visible, setVisible] = useState(false);

  async function generatePassword(len = 15) {
    const chars =
      "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*()-_=+[]{}|;:,.<>?";
    let res = "";
    for (let i = 0; i < len; i++) {
      res += chars.charAt(Math.floor(Math.random() * chars.length));
    }
    await navigator.clipboard.writeText(res);
    return res;
  }

  async function handleGenerate(e: React.MouseEvent<HTMLButtonElement>) {
    e.preventDefault();
    e.stopPropagation();
    const generated = await generatePassword();
    if (onGenerate) {
      onGenerate(generated);
      snack.info(t("user_messages.password_generated_message"), {
        duration: 15000,
      });
      return;
    }
    props.onChange?.({
      target: { value: generated },
    } as React.ChangeEvent<HTMLInputElement>);
    snack.info(t("user_messages.password_generated_message"), {
      duration: 15000,
    });
  }

  return (
    <div className="password-input-wrapper">
      <Input
        {...props}
        type={visible ? "text" : "password"}
        className="password-input"
        icon={<LockKeyhole size={18} />}
      />

      <div className="password-input-icons">
        {showGenerateButton && (
          <TextTooltip position={"top"} text={t("common.generate_password")}>
            <Button type="button" onClick={handleGenerate}>
              <KeyRound size={20} className="text-(--text-primary)" />
            </Button>
          </TextTooltip>
        )}

        <TextTooltip
          position={"top"}
          text={visible ? t("common.hide_password") : t("common.show_password")}
        >
          <Button type="button" onClick={() => setVisible((v) => !v)}>
            {visible ? (
              <EyeClosed size={20} className="text-(--text-primary)" />
            ) : (
              <Eye size={20} className="text-(--text-primary)" />
            )}
          </Button>
        </TextTooltip>
      </div>
    </div>
  );
};

export default PasswordInput;

```

## File: connectfy-client/src/components/ui/CustomInput/Input/Input.tsx
```typescript
import "./input.style.css";
import React, { useState, useId, forwardRef } from "react";
import ErrorIcon from "@/assets/icons/ErrorIcon.tsx";

export interface CustomInputProps
  extends React.InputHTMLAttributes<HTMLInputElement> {
  isFloating?: boolean;
  isError?: boolean;
  error?: string;
  title?: string;
  inputSize?: "small" | "medium" | "large" | "xlarge";
  icon?: React.ReactNode;
}

export const Input = forwardRef<HTMLInputElement, CustomInputProps>(
  (
    {
      isFloating = true,
      isError = false,
      error,
      title,
      value,
      inputSize = "large",
      className = "",
      icon,
      ...props
    },
    ref,
  ) => {
    const [isFocused, setIsFocused] = useState(false);
    const inputId = useId();

    const isLabelFloating =
      isFloating && (isFocused || (value && String(value).length > 0));

    return (
      <div className="input-wrapper">
        <div
          className={`input-container ${isFloating ? "floating" : ""} ${isError ? "error" : ""} input-size-${inputSize}`}
        >
          {isFloating && title && (
            <label
              htmlFor={inputId}
              className={`floating-label ${isLabelFloating ? "active" : ""}`}
            >
              {title}
            </label>
          )}

          {!isFloating && title && (
            <label htmlFor={inputId} className="static-label">
              {title}
            </label>
          )}

          {icon && <div className="input-icon">{icon}</div>}

          <input
            {...props}
            ref={ref}
            id={inputId}
            value={value}
            className={`input-field ${className}`}
            onFocus={(e) => {
              setIsFocused(true);
              props.onFocus?.(e);
            }}
            onBlur={(e) => {
              setIsFocused(false);
              props.onBlur?.(e);
            }}
            aria-invalid={isError}
            aria-describedby={isError ? `${inputId}-error` : undefined}
            autoComplete="off"
          />
        </div>

        {/* Error wrapper with fixed height to prevent layout shift */}
        {isError && error && (
          <div className="error-wrapper">
            <div
              className={`error-container ${isError && error ? "visible" : ""}`}
              id={`${inputId}-error`}
              role="alert"
              aria-live="polite"
            >
              <ErrorIcon />
              <p className="error-text">{error}</p>
            </div>
          </div>
        )}
      </div>
    );
  },
);

Input.displayName = "Input";

export default Input;

```

## File: connectfy-client/src/components/ui/CustomTextArea/TextArea/Textarea.tsx
```typescript
import "./textarea.style.css";
import React, { useState, useId, forwardRef } from "react";
import ErrorIcon from "@/assets/icons/ErrorIcon.tsx";

export interface CustomTextAreaProps
  extends React.TextareaHTMLAttributes<HTMLTextAreaElement> {
  isFloating?: boolean;
  isError?: boolean;
  error?: string;
  title?: string;
  showCharCount?: boolean;
}

export const Textarea = forwardRef<HTMLTextAreaElement, CustomTextAreaProps>(
  (
    {
      isFloating = true,
      isError = false,
      error,
      title,
      value,
      maxLength,
      showCharCount = false,
      className = "",
      ...props
    },
    ref,
  ) => {
    const [isFocused, setIsFocused] = useState(false);
    const textareaId = useId();

    const currentLength = value ? String(value).length : 0;
    const isLabelFloating = isFloating && (isFocused || currentLength > 0);
    const isLimitReached =
      maxLength !== undefined && currentLength >= maxLength;

    return (
      <div className="textarea-wrapper">
        {!isFloating && title && (
          <label htmlFor={textareaId} className="textarea-static-label">
            {title}
          </label>
        )}

        <div
          className={`textarea-container ${isFloating ? "floating" : ""} ${isError ? "error" : ""}`}
        >
          {isFloating && title && (
            <label
              htmlFor={textareaId}
              className={`textarea-floating-label ${isLabelFloating ? "active" : ""}`}
            >
              {title}
            </label>
          )}

          <textarea
            {...props}
            ref={ref}
            id={textareaId}
            value={value}
            maxLength={maxLength}
            className={`textarea-field ${className}`}
            onFocus={(e) => {
              setIsFocused(true);
              props.onFocus?.(e);
            }}
            onBlur={(e) => {
              setIsFocused(false);
              props.onBlur?.(e);
            }}
            aria-invalid={isError}
            aria-describedby={isError ? `${textareaId}-error` : undefined}
          />
        </div>

        <div className="textarea-footer">
          {showCharCount && maxLength !== undefined && (
            <span
              className={`textarea-char-count ${isLimitReached ? "limit-reached" : ""}`}
            >
              {currentLength}/{maxLength}
            </span>
          )}
        </div>

        {isError && error && (
          <div className="textarea-error-wrapper">
            <div
              className={`textarea-error-container ${isError && error ? "visible" : ""}`}
              id={`${textareaId}-error`}
              role="alert"
              aria-live="polite"
            >
              <ErrorIcon />
              <p className="textarea-error-text">{error}</p>
            </div>
          </div>
        )}
      </div>
    );
  },
);

Textarea.displayName = "Textarea";

export default Textarea;

```

## File: connectfy-client/src/components/ui/CustomTextArea/TextEditor/RichTextEditor.tsx
```typescript
import { useEffect } from "react";
import { useEditor, EditorContent } from "@tiptap/react";
import StarterKit from "@tiptap/starter-kit";
import { Bold, Italic, List, ListOrdered, Strikethrough } from "lucide-react";

interface IProps {
  value: string;
  onChange: (val: string) => void;
  placeholder?: string;
  maxLenght?: number;
}

const RichTextEditor = ({
  value,
  onChange,
  placeholder,
  maxLenght = 100,
}: IProps) => {
  const editor = useEditor({
    extensions: [StarterKit],
    content: value,
    onUpdate: ({ editor }) => {
      onChange(editor.getHTML());
    },
  });

  useEffect(() => {
    if (!editor) return;
    const currentHTML = editor.getHTML();
    if (currentHTML !== value) {
      editor.commands.setContent(value || "");
    }
  }, [value, editor]);

  if (!editor) return null;

  const charCount = editor.getText().length;
  const isOverLimit = charCount > maxLenght;

  const tools = [
    {
      icon: <Bold size={16} />,
      action: () => editor.chain().focus().toggleBold().run(),
      active: editor.isActive("bold"),
    },
    {
      icon: <Italic size={16} />,
      action: () => editor.chain().focus().toggleItalic().run(),
      active: editor.isActive("italic"),
    },
    {
      icon: <Strikethrough size={16} />,
      action: () => editor.chain().focus().toggleStrike().run(),
      active: editor.isActive("strike"),
    },
    {
      icon: <List size={16} />,
      action: () => editor.chain().focus().toggleBulletList().run(),
      active: editor.isActive("bulletList"),
    },
    {
      icon: <ListOrdered size={16} />,
      action: () => editor.chain().focus().toggleOrderedList().run(),
      active: editor.isActive("orderedList"),
    },
  ];

  return (
    <div className="flex flex-col gap-1">
      <div className="border border-(--auth-glass-border) rounded-xl overflow-hidden">
        {/* Toolbar */}
        <div className="flex items-center gap-1 p-2 border-b border-(--auth-glass-border) bg-black/5 dark:bg-white/5">
          {tools.map((tool, i) => (
            <button
              key={i}
              type="button"
              onClick={tool.action}
              className={`p-2 rounded-lg transition-all ${
                tool.active
                  ? "bg-(--primary-color) text-white"
                  : "text-(--muted-color) hover:bg-black/10 dark:hover:bg-white/10"
              }`}
            >
              {tool.icon}
            </button>
          ))}
        </div>

        <div className="relative">
          <EditorContent
            editor={editor}
            className="min-h-[120px] p-3 text-(--text-primary) text-sm prose prose-sm max-w-none [&_.ProseMirror]:outline-none [&_.ProseMirror]:min-h-[100px]"
          />
          {!editor.getText() && (
            <p className="absolute top-3 left-3 pointer-events-none text-(--muted-color) text-sm">
              {placeholder}
            </p>
          )}
        </div>
      </div>

      <p
        className={`text-xs text-right pr-1 ${isOverLimit ? "text-red-500" : "text-(--muted-color)"}`}
      >
        {charCount} / {maxLenght}
      </p>
    </div>
  );
};

export default RichTextEditor;

```

## File: connectfy-client/src/components/ui/Typography/Typography.tsx
```typescript
import React from "react";
import clsx from "clsx";
import "./typography.css";

type Variant =
  | "display-lg"
  | "display-md"
  | "h1"
  | "h2"
  | "h3"
  | "body-lg"
  | "body-md"
  | "body-sm"
  | "caption";

type TypographyProps<T extends React.ElementType> = {
  as?: T;
  variant?: Variant;
  className?: string;
  children: React.ReactNode;
} & React.ComponentPropsWithoutRef<T>;

export function Typography<T extends React.ElementType = "p">({
  as,
  variant = "body-md",
  className,
  children,
  ...rest
}: TypographyProps<T>) {
  const Component = as || "p";

  return (
    <Component
      className={clsx("typography", `typography--${variant}`, className)}
      {...rest}
    >
      {children}
    </Component>
  );
}

```

## File: connectfy-client/src/components/ui/CustomToast/CustomToast.tsx
```typescript
import toast from "react-hot-toast";
import Button from "../CustomButton/Button/Button";
import {
  BellRing,
  CheckCircle,
  CircleAlert,
  Info,
  LucideProps,
  TriangleAlert,
  X,
} from "lucide-react";
import { ForwardRefExoticComponent, RefAttributes } from "react";

type ToastType = "success" | "error" | "warning" | "info" | "default";

const CONFIG: Record<
  ToastType,
  {
    icon: ForwardRefExoticComponent<
      Omit<LucideProps, "ref"> & RefAttributes<SVGSVGElement>
    >;
    bar: string;
    iconClass: string;
  }
> = {
  success: {
    icon: CheckCircle,
    bar: "bg-(--primary-color)",
    iconClass: "text-(--primary-color)",
  },
  error: {
    icon: CircleAlert, // və ya "cancel"
    bar: "bg-(--error-color)",
    iconClass: "text-(--error-color)",
  },
  warning: {
    icon: TriangleAlert,
    bar: "bg-amber-400",
    iconClass: "text-amber-400",
  },
  info: {
    icon: Info,
    bar: "bg-blue-400",
    iconClass: "text-blue-400",
  },
  default: {
    icon: BellRing,
    bar: "bg-(--muted-color)",
    iconClass: "text-(--muted-color)",
  },
};

type Props = {
  toastId: string;
  message: string;
  type: ToastType;
  visible: boolean;
};

const CustomToast = ({ toastId, message, type, visible }: Props) => {
  const { icon: Icon, bar, iconClass } = CONFIG[type];

  return (
    <div
      className={`
        relative flex items-center gap-3
        min-w-[280px] max-w-[420px]
        bg-(--card-bg) text-(--text-color)
        rounded-xl overflow-hidden
        shadow-(--card-shadow)
        border border-(--input-border)
        px-4 py-3
        transition-all duration-300 ease-in-out
        ${visible ? "opacity-100 translate-y-0" : "opacity-0 translate-y-2"}
      `}
    >
      {/* Left accent bar */}
      <span className={`absolute left-0 top-0 h-full w-[3px] ${bar}`} />

      <Icon size={20} className={`shrink-0 ${iconClass}`} />

      {/* Message (Responsive text size for desktop) */}
      <p className="flex-1 text-sm md:text-base leading-snug">{message}</p>

      {/* Close */}
      <Button
        onClick={() => toast.dismiss(toastId)}
        className="
          shrink-0 p-1 flex items-center justify-center rounded-md
          text-(--muted-color)
          hover:bg-(--active-bg)
          hover:text-(--text-color)
          transition-colors duration-150
        "
        aria-label="Close"
        icon={<X size={18} />}
      />
    </div>
  );
};

export default CustomToast;

```

## File: connectfy-client/src/components/ui/CustomToast/FriendshipToast.tsx
```typescript
import toast from "react-hot-toast";
import { Check, X } from "lucide-react";
import Button from "../CustomButton/Button/Button";
import { NotificationType } from "@/common/enums/enums";
import { Fragment } from "react";
import { useAppNavigation } from "@/hooks/useAppNavigation";
import { ROUTER } from "@/common/constants/routet";

export interface IFriendshipToastProps {
  toastId: string;
  actorUsername: string;
  actorAvatar: string | null;
  title: string | null;
  body: string | null;
  isLoading: boolean;
  acceptLabel: string;
  declineLabel: string;
  onAccept: () => void;
  onDecline: () => void;
  type: NotificationType;
  visible: boolean;
  showBody?: boolean;
}

const FriendshipToast = ({
  toastId,
  actorUsername,
  actorAvatar,
  title,
  body,
  isLoading,
  acceptLabel,
  declineLabel,
  onAccept,
  onDecline,
  type,
  visible,
  showBody = true,
}: IFriendshipToastProps) => {
  const initials = actorUsername?.slice(0, 2).toUpperCase() ?? "??";
  const { navigate } = useAppNavigation();

  const handleAccept = (e: React.MouseEvent) => {
    e.stopPropagation();
    onAccept();
    toast.dismiss(toastId);
  };

  const handleDecline = (e: React.MouseEvent) => {
    e.stopPropagation();
    onDecline();
    toast.dismiss(toastId);
  };

  return (
    <Fragment>
      {visible && (
        <div
          className={`
        relative flex items-start gap-3
        min-w-[280px] max-w-[420px]
        bg-(--card-bg) text-(--text-color)
        rounded-xl overflow-hidden
        shadow-(--card-shadow)
        border border-(--input-border)
        pl-5 pr-3 py-3
        transition-all duration-300 ease-in-out cursor-pointer
      `}
          onClick={() => navigate(ROUTER.NOTIFICATIONS.MAIN)}
        >
          <span className="absolute left-0 top-0 h-full w-[3px] bg-(--primary-color)" />

          {/* Avatar */}
          <div className="shrink-0 w-10 h-10 rounded-full overflow-hidden bg-(--active-bg) flex items-center justify-center">
            {actorAvatar ? (
              <img
                src={actorAvatar}
                alt={actorUsername}
                className="w-full h-full object-cover"
                loading="lazy"
              />
            ) : (
              <span className="text-sm font-medium text-(--primary-color)">
                {initials}
              </span>
            )}
          </div>

          {/* Content */}
          <div className="flex-1 min-w-0">
            {title && (
              <p className="text-sm font-semibold leading-tight truncate mb-0.5">
                {title}
              </p>
            )}
            {showBody && (
              <Fragment>
                {body && (
                  <p className="text-xs text-(--muted-color) leading-snug mb-2.5">
                    {body}
                  </p>
                )}

                {type === NotificationType.FRIENDSHIP_REQUEST_SENT && (
                  <div className="flex items-center gap-2">
                    <Button
                      onClick={handleAccept}
                      disabled={isLoading}
                      isLoading={isLoading}
                      className="
                  flex items-center gap-1.5
                  text-xs font-medium px-3 py-1.5 rounded-md
                  bg-(--primary-color) text-white
                  hover:opacity-90 disabled:opacity-50
                  transition-opacity duration-150 cursor-pointer
                "
                      icon={<Check size={13} />}
                      title={acceptLabel}
                      hideTitleInMobile={false}
                    />

                    <Button
                      onClick={handleDecline}
                      disabled={isLoading}
                      isLoading={isLoading}
                      className="
                  flex items-center gap-1.5
                  text-xs font-medium px-3 py-1.5 rounded-md
                  border border-(--input-border)
                  text-(--muted-color)
                  hover:bg-(--active-bg) hover:text-(--text-color)
                  disabled:opacity-50
                  transition-colors duration-150 cursor-pointer
                "
                      icon={<X size={13} />}
                      title={declineLabel}
                      hideTitleInMobile={false}
                    />
                  </div>
                )}
              </Fragment>
            )}
          </div>

          {/* Close */}
          <Button
            onClick={(e) => {
              e.stopPropagation();
              toast.dismiss(toastId);
            }}
            className="
          shrink-0 p-1 flex items-center justify-center rounded-md
          text-(--muted-color)
          hover:bg-(--active-bg) hover:text-(--text-color)
          transition-colors duration-150 cursor-pointer
        "
            icon={<X size={16} />}
          />
        </div>
      )}
    </Fragment>
  );
};

export default FriendshipToast;

```

## File: connectfy-client/src/components/ui/CustomRadio/RadioGroup.tsx
```typescript
import { FC } from "react";

interface Props {
  options: {
    key: string;
    name: string;
    description: string;
  }[];
  value: string;
  onChange: (name: string, value: string) => void;
  name: string;
}

const RadioGroup: FC<Props> = ({ options, value, onChange, name }) => {
  return (
    <div className="flex flex-col gap-[10px] w-full">
      {options.map((option) => {
        const isActive = value === option.key;

        return (
          <div
            key={option.key}
            onClick={() => onChange(name, option.key)}
            className={`
              flex items-center gap-3 p-4 cursor-pointer rounded-xl border-2 transition-all duration-200
              ${
                isActive
                  ? "bg-emerald-500/10 border-(--third-color)"
                  : "bg-slate-50 dark:bg-white/5 border-transparent hover:bg-slate-100 dark:hover:bg-white/10 hover:border-(--third-color)"
              }
            `}
          >
            {/* Radio Circle */}
            <div
              className={`
              relative w-5 h-5 rounded-full border-2 shrink-0 transition-colors duration-200
              ${isActive ? "border-(--third-color)" : "border-slate-300"}
            `}
            >
              <div
                className={`
                absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 w-2.5 h-2.5 rounded-full bg-(--third-color) transition-transform duration-200
                ${isActive ? "scale-100" : "scale-0"}
              `}
              />
            </div>

            {/* Content */}
            <div className="flex-1 text-left">
              <h4 className="text-sm font-bold text-slate-900 dark:text-slate-50 mb-1">
                {option.name}
              </h4>
              <p className="text-xs text-slate-500 dark:text-slate-400 leading-tight">
                {option.description}
              </p>
            </div>
          </div>
        );
      })}
    </div>
  );
};

export default RadioGroup;

```

## File: connectfy-client/src/components/ui/DatePicker/DatePicker.tsx
```typescript
import React, { useState, useRef, useEffect } from "react";
import { createPortal } from "react-dom";
import "./datePicker.style.css";
import { useTranslation } from "react-i18next";
import DatePickerModal from "../../Modal/DatePickerModal/DatePickerModal";
import Button from "../CustomButton/Button/Button";
import { CalendarDays, ChevronLeft, ChevronRight } from "lucide-react";
import { useIsMobile } from "@/hooks/useIsMobile";

interface CustomDatePickerProps {
  value?: string | Date | null;
  onChange: (date: string) => void;
  hasError?: boolean;
  inputSize?: "small" | "medium" | "large" | "xlarge";
  placeholder?: string;
  onKeyDown?: (e: React.KeyboardEvent<HTMLElement>) => void;
  title?: string;
}

type ViewMode = "days" | "months" | "years";

export default function CustomDatePicker({
  value,
  onChange,
  hasError = false,
  inputSize = "large",
  placeholder,
  onKeyDown,
  title,
}: CustomDatePickerProps) {
  const { t } = useTranslation();

  const isMobile = useIsMobile();
  const [isOpen, setIsOpen] = useState<boolean>(false);
  const [selectedDate, setSelectedDate] = useState<Date | null>(
    value ? (typeof value === "string" ? new Date(value) : value) : null,
  );
  const [currentMonth, setCurrentMonth] = useState<Date>(new Date());
  const [viewMode, setViewMode] = useState<ViewMode>("days");
  const [popupPosition, setPopupPosition] = useState({
    top: 0,
    left: 0,
    width: 0,
  });

  const calendarRef = useRef<HTMLDivElement>(null);
  const popupRef = useRef<HTMLDivElement>(null);

  const minDate = new Date(1960, 0, 1);
  const maxDate = new Date();
  const currentYear = new Date().getFullYear();

  const formatDate = (date: Date | null): string => {
    if (!date) return "";
    const year = date.getFullYear();
    const month = String(date.getMonth() + 1).padStart(2, "0");
    const day = String(date.getDate()).padStart(2, "0");
    return `${year}-${month}-${day}`;
  };

  const displayDate = selectedDate
    ? selectedDate.toLocaleDateString("en-US", {
        month: "2-digit",
        day: "2-digit",
        year: "numeric",
      })
    : "";

  const handleDateSelect = (date: Date) => {
    if (date >= minDate && date <= maxDate) {
      setSelectedDate(date);
      onChange(formatDate(date));
      setIsOpen(false);
      setViewMode("days");
    }
  };

  const handleSelectDateFromModal = (dateString: string) => {
    const date = new Date(dateString);
    setSelectedDate(date);
    onChange(dateString);
  };

  const handleClear = () => {
    setSelectedDate(null);
    setIsOpen(false);
    onChange("");
  };

  const handleMonthSelect = (month: number) => {
    const newDate = new Date(currentMonth);
    newDate.setMonth(month);
    setCurrentMonth(newDate);
    setViewMode("days");
  };

  const handleYearSelect = (year: number) => {
    const newDate = new Date(currentMonth);
    newDate.setFullYear(year);

    if (year >= minDate.getFullYear() && year <= maxDate.getFullYear()) {
      setCurrentMonth(newDate);
      setViewMode("months");
    }
  };

  const handleToday = () => {
    const today = new Date();
    setSelectedDate(today);
    setCurrentMonth(today);
    onChange(formatDate(today));
    setIsOpen(false);
    setViewMode("days");
  };

  const navigateMonth = (direction: "prev" | "next") => {
    setCurrentMonth((prev) => {
      const newDate = new Date(prev);
      if (direction === "prev") {
        if (viewMode === "years") {
          const newYear = prev.getFullYear() - 12;
          if (newYear >= minDate.getFullYear()) {
            newDate.setFullYear(newYear);
          }
        } else {
          newDate.setMonth(prev.getMonth() - 1);
        }
      } else {
        if (viewMode === "years") {
          const newYear = prev.getFullYear() + 12;
          if (newYear <= currentYear) {
            newDate.setFullYear(newYear);
          }
        } else {
          newDate.setMonth(prev.getMonth() + 1);
        }
      }
      return newDate;
    });
  };

  const getDaysInMonth = (date: Date) => {
    const year = date.getFullYear();
    const month = date.getMonth();
    const firstDay = new Date(year, month, 1);
    const lastDay = new Date(year, month + 1, 0);
    const days = [];

    for (let i = 0; i < firstDay.getDay(); i++) {
      days.push(null);
    }

    for (let i = 1; i <= lastDay.getDate(); i++) {
      days.push(new Date(year, month, i));
    }

    return days;
  };

  const getMonths = () => {
    const months = [];
    for (let i = 0; i < 12; i++) {
      months.push(i);
    }
    return months;
  };

  const getYears = () => {
    const currentYear = currentMonth.getFullYear();
    const startYear = Math.floor(currentYear / 12) * 12;
    const years = [];

    for (let i = startYear - 1; i < startYear + 14; i++) {
      years.push(i);
    }
    return years;
  };

  const isDateDisabled = (date: Date | null): boolean => {
    if (!date) return false;
    return date < minDate || date > maxDate;
  };

  const isMonthDisabled = (month: number): boolean => {
    const testDate = new Date(currentMonth.getFullYear(), month, 15);
    return testDate < minDate || testDate > maxDate;
  };

  const isYearDisabled = (year: number): boolean => {
    return year < minDate.getFullYear() || year > maxDate.getFullYear();
  };

  const days = getDaysInMonth(currentMonth);
  const months = getMonths();
  const years = getYears();
  const today = new Date();

  const weekDays = [
    t("calendar.days.sun"),
    t("calendar.days.mon"),
    t("calendar.days.tue"),
    t("calendar.days.wed"),
    t("calendar.days.thu"),
    t("calendar.days.fri"),
    t("calendar.days.sat"),
  ];

  const monthShortNames = [
    "jan",
    "feb",
    "mar",
    "apr",
    "may",
    "jun",
    "jul",
    "aug",
    "sep",
    "oct",
    "nov",
    "dec",
  ];
  const monthFullNames = [
    "january",
    "february",
    "march",
    "april",
    "may_full",
    "june",
    "july",
    "august",
    "september",
    "october",
    "november",
    "december",
  ];

  const renderHeader = () => {
    if (viewMode === "years") {
      const startYear = years[0];
      const endYear = years[years.length - 1];
      return `${startYear} - ${endYear}`;
    }
    if (viewMode === "months") {
      return currentMonth.getFullYear().toString();
    }
    const monthIndex = currentMonth.getMonth();
    const monthKey = monthFullNames[monthIndex];
    const monthName = t(`calendar.months.${monthKey}`);
    const year = currentMonth.getFullYear();
    return `${monthName} ${year}`;
  };

  const handleHeaderClick = () => {
    if (viewMode === "days") setViewMode("months");
    else if (viewMode === "months") setViewMode("years");
  };

  const isPrevDisabled = () => {
    if (viewMode === "years") return years[0] <= minDate.getFullYear();
    if (viewMode === "months")
      return currentMonth.getFullYear() <= minDate.getFullYear();
    const prevMonth = new Date(currentMonth);
    prevMonth.setMonth(currentMonth.getMonth() - 1);
    return prevMonth < minDate;
  };

  const isNextDisabled = () => {
    if (viewMode === "years")
      return years[years.length - 1] >= maxDate.getFullYear();
    if (viewMode === "months")
      return currentMonth.getFullYear() >= maxDate.getFullYear();
    const nextMonth = new Date(currentMonth);
    nextMonth.setMonth(currentMonth.getMonth() + 1);
    return nextMonth > maxDate;
  };

  useEffect(() => {
    function handleClickOutside(event: MouseEvent) {
      if (isMobile) return;
      const target = event.target as Node;
      if (
        calendarRef.current &&
        !calendarRef.current.contains(target) &&
        popupRef.current &&
        !popupRef.current.contains(target)
      ) {
        setIsOpen(false);
        setViewMode("days");
      }
    }
    document.addEventListener("mousedown", handleClickOutside);
    return () => document.removeEventListener("mousedown", handleClickOutside);
  }, [isMobile]);

  useEffect(() => {
    if (value)
      setSelectedDate(typeof value === "string" ? new Date(value) : value);
    else setSelectedDate(null);
  }, [value]);

  useEffect(() => {
    if (isOpen && calendarRef.current && !isMobile) {
      const rect = calendarRef.current.getBoundingClientRect();
      setPopupPosition({
        top: rect.bottom + window.scrollY,
        left: rect.left + window.scrollX,
        width: rect.width,
      });
    }
  }, [isOpen, isMobile]);

  const handleModalClose = () => {
    setIsOpen(false);
    setViewMode("days");
  };

  return (
    <>
      <div
        className={`custom_date_picker ${inputSize} ${hasError ? "error" : ""}`}
        ref={calendarRef}
      >
        <div
          className="date_input"
          onClick={() => {
            if (!isOpen && selectedDate) {
              setCurrentMonth(selectedDate);
            }
            setIsOpen(!isOpen);
          }}
        >
          {/* İKON ARTIQ SOLDA VƏ İNPUTDAN ÖNCƏDİR */}
          <div className="calendar_icon">
            <CalendarDays />
          </div>
          {title && (
            <span
              className={`date_picker_label ${isOpen || selectedDate ? "active" : ""}`}
            >
              {title}
            </span>
          )}
          <input
            type="text"
            readOnly
            value={displayDate}
            placeholder={placeholder}
            className="date_display"
            onKeyDown={(e) => (onKeyDown ? onKeyDown(e) : undefined)}
          />
        </div>

        {isOpen &&
          !isMobile &&
          createPortal(
            <div
              ref={popupRef}
              className="calendar_popup"
              style={{
                position: "fixed",
                top: popupPosition.top,
                left: popupPosition.left,
                width: popupPosition.width,
                zIndex: 9999,
              }}
            >
              <div className="calendar_header">
                <Button
                  className={`nav_button ${isPrevDisabled() ? "disabled" : ""}`}
                  onClick={() => !isPrevDisabled() && navigateMonth("prev")}
                  disabled={isPrevDisabled()}
                  type="button"
                  icon={<ChevronLeft size={20} />}
                />
                <div className="current_period" onClick={handleHeaderClick}>
                  {renderHeader()}
                </div>
                <Button
                  className={`nav_button ${isNextDisabled() ? "disabled" : ""}`}
                  onClick={() => !isNextDisabled() && navigateMonth("next")}
                  disabled={isNextDisabled()}
                  type="button"
                  icon={<ChevronRight size={20} />}
                />
              </div>

              {viewMode === "days" && (
                <>
                  <div className="week_days">
                    {weekDays.map((day) => (
                      <div key={day} className="week_day">
                        {day}
                      </div>
                    ))}
                  </div>
                  <div className="calendar_grid">
                    {days.map((date, index) => (
                      <div
                        key={index}
                        className={`calendar_day ${date ? "" : "empty"} ${
                          date &&
                          selectedDate &&
                          date.toDateString() === selectedDate.toDateString()
                            ? "selected"
                            : ""
                        } ${date && date.toDateString() === today.toDateString() ? "today" : ""} ${
                          date && date.getMonth() !== currentMonth.getMonth()
                            ? "other_month"
                            : ""
                        } ${date && isDateDisabled(date) ? "disabled" : ""}`}
                        onClick={() =>
                          date &&
                          !isDateDisabled(date) &&
                          handleDateSelect(date)
                        }
                      >
                        {date ? date.getDate() : ""}
                      </div>
                    ))}
                  </div>
                </>
              )}

              {viewMode === "months" && (
                <div className="months_grid">
                  {months.map((month) => (
                    <div
                      key={month}
                      className={`calendar_month ${month === currentMonth.getMonth() ? "selected" : ""} ${isMonthDisabled(month) ? "disabled" : ""}`}
                      onClick={() =>
                        !isMonthDisabled(month) && handleMonthSelect(month)
                      }
                    >
                      {t(`calendar.months.${monthShortNames[month]}`)}
                    </div>
                  ))}
                </div>
              )}

              {viewMode === "years" && (
                <div className="years_grid">
                  {years.map((year) => (
                    <div
                      key={year}
                      className={`calendar_year ${year === currentMonth.getFullYear() ? "selected" : ""} ${isYearDisabled(year) ? "disabled" : ""}`}
                      onClick={() =>
                        !isYearDisabled(year) && handleYearSelect(year)
                      }
                    >
                      {year}
                    </div>
                  ))}
                </div>
              )}

              <div className="calendar_footer">
                <Button
                  className="footer_button"
                  onClick={handleClear}
                  type="button"
                >
                  {t("common.clear")}
                </Button>
                <Button
                  className="footer_button today_button"
                  onClick={handleToday}
                  type="button"
                >
                  {t("common.today")}
                </Button>
              </div>
            </div>,
            document.body,
          )}
      </div>

      {isMobile && (
        <DatePickerModal
          open={isOpen}
          onClose={handleModalClose}
          onSelectDate={handleSelectDateFromModal}
          onClear={handleClear}
          selectedDate={selectedDate}
          currentMonth={currentMonth}
          setCurrentMonth={setCurrentMonth}
          viewMode={viewMode}
          setViewMode={setViewMode}
          minDate={minDate}
          maxDate={maxDate}
        />
      )}
    </>
  );
}

```

## File: connectfy-client/src/components/ui/CustomButton/Button/Button.tsx
```typescript
import TextTooltip from "@/components/Tooltip/TextTooltip";
import React, { FC, Fragment, ReactNode } from "react";

export interface CustomButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  title?: string;
  icon?: ReactNode;
  isLoading?: boolean;
  loadingText?: string;
  tooltip?: string;
  children?: ReactNode;
  tooltipPosition?: "top" | "bottom" | "left" | "right";
  hideTitleInMobile?: boolean;
}

const Button: FC<CustomButtonProps> = ({
  title,
  icon,
  isLoading = false,
  loadingText,
  tooltip,
  disabled,
  className = "",
  children,
  tooltipPosition,
  hideTitleInMobile = true,
  ...props
}) => {
  const titleClassName = icon && hideTitleInMobile ? "hidden sm:inline" : "";

  const content = children ? (
    children
  ) : (
    <div className="flex items-center justify-center gap-2">
      {isLoading ? (
        <Fragment>
          <svg className="w-5 h-5 animate-spin" viewBox="0 0 24 24" fill="none">
            <circle
              className="opacity-25"
              cx="12"
              cy="12"
              r="10"
              stroke="currentColor"
              strokeWidth="4"
            />
            <path
              className="opacity-75"
              fill="currentColor"
              d="M4 12a8 8 0 018-8v3a5 5 0 00-5 5H4z"
            />
          </svg>
          {loadingText && <span>{loadingText}</span>}
        </Fragment>
      ) : icon ? (
        <Fragment>
          <span aria-hidden="true">{icon}</span>
          {title && <span className={titleClassName}>{title}</span>}
        </Fragment>
      ) : (
        <Fragment>
          {title && <span className={titleClassName}>{title}</span>}
        </Fragment>
      )}
    </div>
  );

  const buttonElement = (
    <button
      {...props}
      disabled={disabled || isLoading}
      aria-busy={isLoading}
      className={`${className} ${disabled || isLoading ? "opacity-70" : ""} ${
        isLoading
          ? "cursor-wait"
          : disabled
            ? "cursor-not-allowed shadow-none!"
            : "cursor-pointer"
      }`}
    >
      {content}
    </button>
  );

  if (!tooltip) {
    return buttonElement;
  }

  return (
    <TextTooltip text={tooltip} position={tooltipPosition}>
      <span className="inline-flex">{buttonElement}</span>
    </TextTooltip>
  );
};

export default Button;

```

## File: connectfy-client/src/components/ui/CustomButton/CloseButton/CloseButton.tsx
```typescript
import { CSSProperties, FC } from "react";
import Button from "../Button/Button";
import { X } from "lucide-react";

interface Props {
  onClick: () => void;
  style?: CSSProperties;
  className?: string;
}

const CloseButton: FC<Props> = ({ onClick, style }) => (
  <Button
    onClick={onClick}
    style={{
      ...style,
      background: "none",
      border: "none",
      cursor: "pointer",
      padding: "4px 8px",
      display: "flex",
      alignItems: "center",
      justifyContent: "center",
      color: "inherit",
      opacity: 0.8,
      transition: "opacity 0.3s ease-in-out",
    }}
    onMouseEnter={(e) => (e.currentTarget.style.opacity = "1")}
    onMouseLeave={(e) => (e.currentTarget.style.opacity = "0.8")}
  >
    <X size={18} />
  </Button>
);

export default CloseButton;

```

## File: connectfy-client/src/components/ContextMenu/ContextMenuRenderer.tsx
```typescript
import { useLayoutEffect, useRef, useState, useCallback } from "react";
import { createPortal } from "react-dom";
import { motion } from "framer-motion";

const ContextMenuRenderer = ({ x, y, content, isOpen, onClose }: any) => {
  const menuRef = useRef<HTMLDivElement>(null);
  const [pos, setPos] = useState({ top: y, left: x, origin: "top left" });
  const [isMeasured, setIsMeasured] = useState(false);

  // Pozisiyanı hesablamaq üçün funksiyanı ayırırıq
  const updatePosition = useCallback(() => {
    if (!menuRef.current) return;

    const menuWidth = menuRef.current.offsetWidth;
    const menuHeight = menuRef.current.offsetHeight;
    const screenWidth = window.innerWidth;
    const screenHeight = window.innerHeight;

    let finalX = x;
    let finalY = y;
    let originX = "left";
    let originY = "top";

    // Sağ tərəf yoxlaması
    if (x + menuWidth > screenWidth) {
      finalX = x - menuWidth;
      originX = "right";
    }

    // Aşağı tərəf yoxlaması
    // Əsas məqam buradır: Hündürlük artdıqca yuxarı doğru sürüşməlidir
    if (y + menuHeight > screenHeight) {
      finalY = screenHeight - menuHeight - 10; // Ekranın altına 10px qalmış dayansın
      originY = "bottom";
    }

    setPos({ top: finalY, left: finalX, origin: `${originY} ${originX}` });
    setIsMeasured(true);
  }, [x, y]);

  useLayoutEffect(() => {
    if (isOpen && menuRef.current) {
      // Ölçü dəyişikliklərini izləyirik (Sub-menuya keçəndə hündürlük dəyişir)
      const resizeObserver = new ResizeObserver(() => {
        updatePosition();
      });

      resizeObserver.observe(menuRef.current);

      return () => resizeObserver.disconnect();
    } else {
      setIsMeasured(false);
    }
  }, [isOpen, updatePosition]);

  if (!isOpen) return null;

  return createPortal(
    <div
      className="fixed inset-0 z-9999"
      onClick={onClose}
      onContextMenu={(e) => {
        e.preventDefault();
        onClose();
      }}
    >
      <motion.div
        ref={menuRef}
        initial="hidden"
        animate={isMeasured ? "visible" : "hidden"}
        exit="hidden"
        variants={{
          hidden: {
            opacity: 0,
            scale: 0.8,
            transition: { duration: 0.15 },
          },
          visible: {
            opacity: 1,
            scale: 1,
            transition: {
              type: "spring",
              stiffness: 300,
              damping: 25,
            },
          },
        }}
        style={{
          position: "absolute",
          top: pos.top,
          left: pos.left,
          transformOrigin: pos.origin,
          visibility: isMeasured ? "visible" : "hidden",
          pointerEvents: "auto",
        }}
        onClick={(e) => e.stopPropagation()}
      >
        {content}
      </motion.div>
    </div>,
    document.body,
  );
};

export default ContextMenuRenderer;

```

## File: connectfy-client/src/components/ContextMenu/ContextMenuItem.tsx
```typescript
import { useContextMenu } from "@/hooks/useContextMenu";
import { FC } from "react";
import Button from "../ui/CustomButton/Button/Button";

interface IProps {
  icon?: any;
  label: string | React.ReactNode;
  onClick: () => void;
  isWarning?: boolean;
  closeAfterClick?: boolean;
  className?: string;
  labelClassName?: string;
  iconClassName?: string;
}

const ContextMenuItem: FC<IProps> = ({
  icon: Icon,
  label,
  onClick,
  isWarning,
  closeAfterClick = true,
  className,
  labelClassName,
  iconClassName,
}) => {
  const { closeMenu } = useContextMenu();

  return (
    <Button
      onClick={(e) => {
        e.stopPropagation();
        onClick();
        if (closeAfterClick) closeMenu();
      }}
      className={`w-full flex items-center justify-between px-4 py-3 text-sm transition-colors hover:bg-white/80 dark:hover:bg-white/5 first:rounded-t-xl last:rounded-b-xl duration-300
        ${isWarning ? "text-red-500" : "text-(--text-color)"} ${className}`}
    >
      <span className={`font-medium ${labelClassName}`}>{label}</span>
      {Icon && (
        <Icon
          size={18}
          className={`${isWarning ? "text-red-500" : "text-(--text-color)"} ${iconClassName}`}
        />
      )}
    </Button>
  );
};

export default ContextMenuItem;

```

## File: connectfy-client/src/components/ContextMenu/Sidebar/DesktopSidebarContextMenu.tsx
```typescript
import { FC, useState } from "react";
import { motion, AnimatePresence } from "framer-motion";
import {
  SunMoon,
  ChevronLeft,
  Sun,
  Moon,
  MonitorSmartphone,
  Languages,
  UserCircle,
} from "lucide-react";
import { useTranslation } from "react-i18next";
import ContextMenuItem from "../ContextMenuItem";
import { useEditGeneralSettingsMutation } from "@/modules/settings/GeneralSettings/api/api";
import { useErrors } from "@/hooks/useErrors";
import { useTheme } from "@/context/ThemeContext";
import { LANGUAGE, THEME } from "@/common/enums/enums";
import { useGeneralSettings } from "@/modules/settings/GeneralSettings/hooks/useGeneralSettings";
import { useAvatarModalStore } from "@/store/zustand/useAvatarModalStore";
import { useUser } from "@/context/UserContext";

// Menyular arası keçid üçün tiplər
type MenuView = "main" | "theme" | "language";

const DesktopSidebarContextMenu: FC = () => {
  const { t, i18n } = useTranslation();
  const { showResponseErrors } = useErrors();
  const { theme, toggleTheme } = useTheme();
  const { user } = useUser();

  const { generalSettings } = useGeneralSettings();
  const [updateGeneralSettings] = useEditGeneralSettingsMutation();

  // Hansı menyunun açıq olduğunu izləyən state
  const [currentView, setCurrentView] = useState<MenuView>("main");

  const onOpenShowModal = useAvatarModalStore((state) => state.onOpenShowModal);

  const handleChangeTheme = async (appTheme: THEME) => {
    try {
      if (theme === appTheme || !generalSettings) return;

      toggleTheme(appTheme);
      await updateGeneralSettings({
        _id: generalSettings._id,
        theme: appTheme,
      }).unwrap();
    } catch (error) {
      toggleTheme(generalSettings?.theme as THEME);
      showResponseErrors(error);
    }
  };

  const handleChangeLanguage = async (language: LANGUAGE) => {
    try {
      if (language === generalSettings?.language || !generalSettings) return;

      i18n.changeLanguage(language);
      await updateGeneralSettings({
        _id: generalSettings._id,
        language,
      }).unwrap();
    } catch (error) {
      i18n.changeLanguage(generalSettings?.language as LANGUAGE);
      showResponseErrors(error);
    }
  };

  // Alt-menyulara keçid animasiyaları
  const viewVariants = {
    initial: { opacity: 0, x: 20 },
    animate: { opacity: 1, x: 0 },
    exit: { opacity: 0, x: -20 },
  };

  return (
    <motion.div
      initial={{ opacity: 0, scale: 0.9, y: -10 }}
      animate={{ opacity: 1, scale: 1, y: 0 }}
      exit={{ opacity: 0, scale: 0.9, y: -10 }}
      transition={{ duration: 0.15, ease: "easeOut" }}
      className="flex flex-col min-w-[220px] bg-(--bg-color) rounded-2xl shadow-2xl overflow-hidden"
    >
      <AnimatePresence mode="wait">
        {/* ƏSAS MENYU */}
        {currentView === "main" && (
          <motion.div
            key="main"
            initial="initial"
            animate="animate"
            exit="exit"
            variants={viewVariants}
            transition={{ duration: 0.2 }}
            className="flex flex-col w-full"
          >
            <ContextMenuItem
              icon={SunMoon}
              label={t("common.change_theme")}
              onClick={() => setCurrentView("theme")}
              closeAfterClick={false}
            />
            <ContextMenuItem
              icon={Languages}
              label={t("common.change_lang")}
              onClick={() => setCurrentView("language")}
              closeAfterClick={false}
            />
            <ContextMenuItem
              icon={UserCircle}
              label={t("common.profile_photo")}
              onClick={() => {
                if (!user?.avatar?.url) return;

                onOpenShowModal(user?.avatar?.url, user?.username, user?._id);
              }}
            />
          </motion.div>
        )}

        {/* THEME MENYUSU */}
        {currentView === "theme" && (
          <motion.div
            key="theme"
            initial="initial"
            animate="animate"
            exit="exit"
            variants={viewVariants}
            transition={{ duration: 0.2 }}
            className="flex flex-col w-full"
          >
            <div className="border-b border-white/10 mb-1 pb-1">
              <ContextMenuItem
                icon={ChevronLeft}
                label={t("common.back", "Back")}
                onClick={() => setCurrentView("main")}
                closeAfterClick={false}
              />
            </div>

            <ContextMenuItem
              icon={Sun}
              label={
                <div className="flex items-center justify-between w-full">
                  {t(`enum.${THEME.LIGHT}`)}
                </div>
              }
              onClick={() => handleChangeTheme(THEME.LIGHT)}
              className={
                theme === THEME.LIGHT
                  ? "bg-(--primary-color)/70 text-white! font-semibold hover:bg-(--primary-color)/70!"
                  : undefined
              }
              iconClassName={theme === THEME.LIGHT ? "text-white!" : undefined}
              closeAfterClick={false}
            />
            <ContextMenuItem
              icon={Moon}
              label={
                <div className={`flex items-center justify-between w-full`}>
                  {t(`enum.${THEME.DARK}`)}
                </div>
              }
              onClick={() => handleChangeTheme(THEME.DARK)}
              className={
                theme === THEME.DARK
                  ? "bg-(--primary-color)/70 text-white! font-semibold hover:bg-(--primary-color)/70!"
                  : undefined
              }
              iconClassName={theme === THEME.DARK ? "text-white!" : undefined}
              closeAfterClick={false}
            />
            <ContextMenuItem
              icon={MonitorSmartphone}
              label={
                <div className="flex items-center justify-between w-full">
                  {t(`enum.${THEME.DEVICE}`)}
                </div>
              }
              onClick={() => handleChangeTheme(THEME.DEVICE)}
              className={
                theme === THEME.DEVICE
                  ? "bg-(--primary-color)/70 text-white! font-semibold hover:bg-(--primary-color)/70!"
                  : undefined
              }
              iconClassName={theme === THEME.DEVICE ? "text-white!" : undefined}
              closeAfterClick={false}
            />
          </motion.div>
        )}

        {/* LANGUAGE MENYUSU */}
        {currentView === "language" && (
          <motion.div
            key="language"
            initial="initial"
            animate="animate"
            exit="exit"
            variants={viewVariants}
            transition={{ duration: 0.2 }}
            className="flex flex-col w-full"
          >
            <div className="border-b border-white/10 mb-1 pb-1">
              <ContextMenuItem
                icon={ChevronLeft}
                label={t("common.back", "Back")}
                onClick={() => setCurrentView("main")}
                closeAfterClick={false}
              />
            </div>

            <ContextMenuItem
              label={
                <div className="flex items-center justify-between w-full">
                  English (EN)
                </div>
              }
              onClick={() => handleChangeLanguage(LANGUAGE.EN)}
              icon={Languages}
              className={
                i18n.language === LANGUAGE.EN
                  ? "bg-(--primary-color)/70 text-white! font-semibold hover:bg-(--primary-color)/70!"
                  : undefined
              }
              iconClassName={
                i18n.language === LANGUAGE.EN ? "text-white!" : undefined
              }
              closeAfterClick={false}
            />
            <ContextMenuItem
              label={
                <div className="flex items-center justify-between w-full">
                  Azərbaycan (AZ)
                </div>
              }
              onClick={() => handleChangeLanguage(LANGUAGE.AZ)}
              icon={Languages}
              className={
                i18n.language === LANGUAGE.AZ
                  ? "bg-(--primary-color)/70 text-white! font-semibold hover:bg-(--primary-color)/70!"
                  : undefined
              }
              iconClassName={
                i18n.language === LANGUAGE.AZ ? "text-white!" : undefined
              }
              closeAfterClick={false}
            />
            <ContextMenuItem
              label={
                <div className="flex items-center justify-between w-full">
                  Русский (RU)
                </div>
              }
              onClick={() => handleChangeLanguage(LANGUAGE.RU)}
              icon={Languages}
              className={
                i18n.language === LANGUAGE.RU
                  ? "bg-(--primary-color)/70 text-white! font-semibold hover:bg-(--primary-color)/70!"
                  : undefined
              }
              iconClassName={
                i18n.language === LANGUAGE.RU ? "text-white!" : undefined
              }
              closeAfterClick={false}
            />
            <ContextMenuItem
              label={
                <div className="flex items-center justify-between w-full">
                  Türkçe (TR)
                </div>
              }
              onClick={() => handleChangeLanguage(LANGUAGE.TR)}
              icon={Languages}
              className={
                i18n.language === LANGUAGE.TR
                  ? "bg-(--primary-color)/70 text-white! font-semibold hover:bg-(--primary-color)/70!"
                  : undefined
              }
              iconClassName={
                i18n.language === LANGUAGE.TR ? "text-white!" : undefined
              }
              closeAfterClick={false}
            />
          </motion.div>
        )}
      </AnimatePresence>
    </motion.div>
  );
};

export default DesktopSidebarContextMenu;

```

## File: connectfy-client/src/components/Header/AuthHeader/AuthHeader.tsx
```typescript
import MainIcon from "@/assets/icons/MainIcon";
import "./authHeader.style.css";

const AuthHeader = () => {
  return (
    <header className="auth-header">
      <div className="logo">
        <MainIcon
          styles={{
            width: "40px",
            height: "40px",
            overflow: "visible",
            opacity: 1,
            zIndex: 1,
            fill: "rgb(46, 204, 113)",
          }}
        />
        <h1>Connectfy</h1>
      </div>
    </header>
  );
};

export default AuthHeader;

```

## File: connectfy-client/src/components/Header/UnqiueHeader/UniqueHeader.tsx
```typescript
import { FC, useRef } from "react";
import { useTranslation } from "react-i18next";
import Button from "@/components/ui/CustomButton/Button/Button";
import { ArrowLeft, Save } from "lucide-react";

interface Props {
  onClickBack: () => void;
  headerTitle: string;
  headerSubtitle: string;
  showHeaderButton?: boolean;
  isHeaderButtonDisabled?: boolean;
  onHeaderButtonClick?: () => void;
  isLoading?: boolean;
}

const UniqueHeader: FC<Props> = ({
  onClickBack,
  headerTitle,
  headerSubtitle,
  showHeaderButton = false,
  isHeaderButtonDisabled = true,
  onHeaderButtonClick,
  isLoading,
}) => {
  const { t } = useTranslation();
  const headerRef = useRef<HTMLDivElement>(null);

  return (
    <div
      ref={headerRef}
      // Arxa plan sənin istədiyin kimi qaldı
      className="
        sticky top-0 z-10 bg-(--bg-color) mb-4
        transition-all duration-300 ease-in-out
        animate-fade-in md:pt-0
      "
    >
      {/* Header Top Row */}
      <div className="flex items-center justify-between gap-4 sm:gap-3">
        {/* Left: Back Button + Title */}
        <div className="flex items-center gap-3 sm:gap-4 flex-1 min-w-0">
          <Button
            type="button"
            aria-label={t("common.back")}
            onClick={onClickBack}
            icon={<ArrowLeft size={20} />}
            className="
              w-[42px] h-[42px] rounded-[10px] border-none shrink-0
              flex items-center justify-center
              bg-(--disabled-bg) text-(--text-primary)
              transition-colors duration-200 ease-in-out
              hover:bg-(--active-bg-2) hover:text-(--primary-color)
              max-[480px]:w-10 max-[480px]:h-10
            "
          />

          <h1
            className="
              text-[24px] md:text-[28px] font-extrabold text-(--text-primary)
              m-0 whitespace-nowrap overflow-hidden text-ellipsis
              tracking-tight
            "
          >
            {headerTitle}
          </h1>
        </div>

        {/* Right: Save Button */}
        {showHeaderButton && (
          <Button
            type="button"
            aria-label={t("common.save_changes")}
            title={t("common.save_changes")}
            disabled={isHeaderButtonDisabled}
            onClick={onHeaderButtonClick}
            isLoading={isLoading}
            icon={<Save size={18} />}
            className={`
              flex items-center gap-2 px-5 shrink-0 h-[42px]
              rounded-[10px] text-white text-[15px] font-semibold border-none
              transition-all duration-300 ease-in-out
              max-[640px]:w-[42px] max-[640px]:p-0 max-[640px]:justify-center
              max-[480px]:w-10 max-[480px]:h-10
              [&>span]:inline-block [&>span]:whitespace-nowrap
              max-[640px]:[&>span]:hidden
              ${
                isHeaderButtonDisabled
                  ? "bg-(--disabled-bg) text-(--disabled-text) shadow-none cursor-not-allowed"
                  : "bg-linear-to-br from-(--third-color) to-(--hover-bg) shadow-(--active-shadow) hover:shadow-lg"
              }
            `}
          />
        )}
      </div>

      {/* Subtitle */}
      <p className="text-[14px] font-medium text-(--muted-color) m-0 mt-2">
        {headerSubtitle}
      </p>
    </div>
  );
};

export default UniqueHeader;

```

## File: connectfy-client/src/components/Tooltip/KeyboardShortcutTooltip.tsx
```typescript
import React from "react";
import { isMac } from "@/common/utils/keyboard";
import { Command, Option } from "lucide-react";

interface ShortcutTooltipProps {
  keys: string[];
  children: React.ReactNode;
  fullWidth?: boolean;
}

export const ShortcutTooltip: React.FC<ShortcutTooltipProps> = ({
  keys,
  children,
  fullWidth = true,
}) => {
  const displayKeys = keys.map((key) => {
    if (key === "Ctrl") {
      return isMac() ? <Command size={14} /> : "Ctrl";
    }
    if (key === "Alt") {
      return isMac() ? <Option size={14} /> : "Alt";
    }
    return key;
  });

  return (
    <div className={`group relative ${fullWidth ? "flex-1" : "inline-block"}`}>
      {children}
      <div className="invisible group-hover:visible opacity-0 group-hover:opacity-100 transition-opacity duration-700 absolute bottom-full left-1/2 -translate-x-1/2 mb-2 px-3 py-2 bg-slate-900 text-white text-xs rounded-lg whitespace-nowrap z-50 pointer-events-none">
        <div className="flex items-center gap-1">
          {displayKeys.map((key, index) => (
            <React.Fragment key={index}>
              {index > 0 && <span className="text-slate-400 mx-0.5">+</span>}
              <kbd className="px-2 py-1.5 text-xs font-bold! text-heading bg-neutral-tertiary border border-default-medium rounded-md">
                {key}
              </kbd>
            </React.Fragment>
          ))}
        </div>
        <div className="absolute top-full left-1/2 -translate-x-1/2 -mt-1">
          <div className="border-4 border-transparent border-t-slate-900" />
        </div>
      </div>
    </div>
  );
};

```

## File: connectfy-client/src/components/Tooltip/TextTooltip.tsx
```typescript
import React, { useState, useRef, useCallback, Fragment } from "react";
import { createPortal } from "react-dom";

interface TextTooltipProps {
  text: string;
  children: React.ReactNode;
  position?: "top" | "bottom" | "left" | "right";
  duration?: number;
}

export const TextTooltip: React.FC<TextTooltipProps> = ({
  text,
  children,
  position = "top",
  duration = 500,
}) => {
  const [isVisible, setIsVisible] = useState(false);
  const [tooltipPosition, setTooltipPosition] = useState({ top: 0, left: 0 });
  const wrapperRef = useRef<HTMLSpanElement>(null);

  const updateTooltipPosition = useCallback(() => {
    if (!wrapperRef.current) return;

    const rect = wrapperRef.current.getBoundingClientRect();
    let top = 0;
    let left = 0;

    switch (position) {
      case "top":
        top = rect.top - 8;
        left = rect.left + rect.width / 2;
        break;
      case "bottom":
        top = rect.bottom + 8;
        left = rect.left + rect.width / 2;
        break;
      case "left":
        top = rect.top + rect.height / 2;
        left = rect.left - 8;
        break;
      case "right":
        top = rect.top + rect.height / 2;
        left = rect.right + 8;
        break;
    }

    setTooltipPosition({ top, left });
  }, [position]);

  const handleMouseEnter = () => {
    updateTooltipPosition();
    setIsVisible(true);
  };

  const handleMouseLeave = () => {
    setIsVisible(false);
  };

  const transformClasses = {
    top: "-translate-x-1/2 -translate-y-full",
    bottom: "-translate-x-1/2",
    left: "-translate-x-full -translate-y-1/2",
    right: "-translate-y-1/2",
  };

  const arrowStyles = {
    top: "top-full left-1/2 -translate-x-1/2 border-l-transparent border-r-transparent border-b-transparent border-t-slate-900",
    bottom:
      "bottom-full left-1/2 -translate-x-1/2 border-l-transparent border-r-transparent border-t-transparent border-b-slate-900",
    left: "left-full top-1/2 -translate-y-1/2 border-t-transparent border-b-transparent border-r-transparent border-l-slate-900",
    right:
      "right-full top-1/2 -translate-y-1/2 border-t-transparent border-b-transparent border-l-transparent border-r-slate-900",
  };

  return (
    <Fragment>
      <span
        ref={wrapperRef}
        onMouseEnter={handleMouseEnter}
        onMouseLeave={handleMouseLeave}
        className="inline-flex"
      >
        {children}
      </span>
      {createPortal(
        <div
          className={`fixed ${transformClasses[position]} px-3 py-2 font-semibold bg-slate-900 text-white text-xs rounded-lg whitespace-nowrap z-999999 pointer-events-none`}
          style={{
            top: tooltipPosition.top,
            left: tooltipPosition.left,
            opacity: isVisible ? 1 : 0,
            transition: `opacity ${duration}ms ease-in-out`,
            visibility: isVisible ? "visible" : "hidden",
            fontFamily:
              "Inter, system-ui, -apple-system, sans-serif !important",
          }}
        >
          {text}
          <div className={`absolute border-4 ${arrowStyles[position]}`} />
        </div>,
        document.body,
      )}
    </Fragment>
  );
};

export default TextTooltip;

```

## File: connectfy-client/src/components/routeGuard/RequireAuth.tsx
```typescript
import { ReactNode, useEffect } from "react";
import { Navigate, useLocation } from "react-router-dom";
import { ROUTER } from "@/common/constants/routet";
import { getHomeRouteByStartup } from "@/common/utils/routes";
import { useAuthStore } from "@/store/zustand/useAuthStore";
import { useUser } from "@/context/UserContext";
import { useGeneralSettings } from "@/modules/settings/GeneralSettings/hooks/useGeneralSettings";
import { useAppNavigation } from "@/hooks/useAppNavigation";

type AuthType = {
  children: ReactNode;
};

export function RequireAuth({ children }: AuthType) {
  const location = useLocation();

  //  Əvvəlcə bütün hooks-ları çağırın
  const { access_token } = useAuthStore();
  useUser();

  // Sonra conditional logic
  if (!access_token) {
    return (
      <Navigate to={ROUTER.AUTH.LOGIN} state={{ from: location }} replace />
    );
  }

  return <>{children}</>;
}

export function InsideProfile({ children }: AuthType) {
  const { access_token } = useAuthStore();

  // Yalnız həqiqətən token varsa request at
  const { isSuccess: isUserSuccess } = useUser();

  const { generalSettings, isSuccess: isSettingsSuccess } =
    useGeneralSettings();

  // Hər iki data uğurla gəlibsə yönləndir
  if (access_token && isUserSuccess && isSettingsSuccess) {
    const startup = generalSettings?.startupPage;
    return <Navigate to={getHomeRouteByStartup(startup)} replace />;
  }

  return <>{children}</>;
}

export function RedirectMain() {
  const { navigate } = useAppNavigation();
  const { access_token } = useAuthStore();

  // Send request if there is a token
  const { isError: isUserError } = useUser();

  const { generalSettings, isSuccess: settingsLoaded } = useGeneralSettings();

  useEffect(() => {
    // If no token, or the user request failed (invalid token), go to login
    if (!access_token || isUserError) {
      navigate(ROUTER.AUTH.LOGIN, { replace: true });
      return;
    }

    // Only navigate once the settings (which contain the startupPage) are ready
    if (settingsLoaded) {
      const path = getHomeRouteByStartup(generalSettings?.startupPage);
      navigate(path, { replace: true });
    }
  }, [access_token, isUserError, settingsLoaded, generalSettings, navigate]);

  return null;
}

```

## File: connectfy-client/src/store/store.ts
```typescript
import { configureStore, combineReducers, Action } from "@reduxjs/toolkit";
import { setupListeners } from "@reduxjs/toolkit/query";

import { authApi } from "@/modules/auth/api/api";
import { profileApi } from "@/modules/profile/api/api";
import { accountSettingsApi } from "@/modules/settings/AccountSettings/api/api";
import { generalSettingsApi } from "@/modules/settings/GeneralSettings/api/api";
import { privacySettingsApi } from "@/modules/settings/PrivacySettings/api/api";
import { notificationSettingsApi } from "@/modules/settings/NotificationSettings/api/api";
import { allUsersApi } from "@/modules/users/AllUsers/api/api";
import { userProfileApi } from "@/modules/users/UserProfile/api/api";
import { myFriendsApi } from "@/modules/users/MyFriends/api/api";
import { notificationApi } from "@/modules/notifications/api/api";
import { blocklistApi } from "@/modules/users/Blocklist/api/api";

export const apis = [
  authApi,
  profileApi,
  accountSettingsApi,
  generalSettingsApi,
  privacySettingsApi,
  notificationSettingsApi,
  allUsersApi,
  userProfileApi,
  myFriendsApi,
  notificationApi,
  blocklistApi,
];

const appReducer = combineReducers(
  Object.fromEntries(apis.map((api) => [api.reducerPath, api.reducer])),
) as any;

export type RootState = ReturnType<typeof appReducer>;

const rootReducer = (state: RootState | undefined, action: Action) => {
  if (action.type === "RESET") {
    state = undefined;
  }
  return appReducer(state, action);
};

export const store = configureStore({
  reducer: rootReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(apis.map((api) => api.middleware)),
});

setupListeners(store.dispatch);

export type AppDispatch = typeof store.dispatch;

```

## File: connectfy-client/src/store/zustand/useAuthStore.ts
```typescript
import { create } from "zustand";

interface AuthState {
  access_token: string | null;
  authenticateToken: string | null;
  setToken: ({
    type,
    token,
  }: {
    type: "access_token" | "authenticateToken";
    token: string;
  }) => void;
  clear: (type: "access_token" | "authenticateToken" | "all") => void;
}

export const useAuthStore = create<AuthState>((set) => ({
  access_token: localStorage.getItem("access_token"),
  authenticateToken: null,

  setToken: ({ type, token }) => {
    if (type === "access_token") {
      localStorage.setItem("access_token", token);
      set({ access_token: token });
    } else {
      set({ authenticateToken: token });
    }
  },

  clear: (type) => {
    if (type === "access_token" || type === "all") {
      localStorage.removeItem("access_token");
      set({ access_token: null });
    }
    if (type === "authenticateToken" || type === "all") {
      set({ authenticateToken: null });
    }
  },
}));

```

## File: connectfy-client/src/store/zustand/useAvatarModalStore.ts
```typescript
import { create } from "zustand";
import { IAvatar, IDefaultAvatar } from "@/modules/profile/types/types";

interface AvatarModalStore {
  // Shared Data
  avatarUrl: string;
  avatarObj: IAvatar | null | undefined;
  username?: string;
  userId?: string;
  profileId: string;
  defaultAvatarObj: IDefaultAvatar | null;

  // Show Avatar Modal
  isShowModalOpen: boolean;
  onOpenShowModal: (
    url: string,
    name?: string,
    userId?: string,
    profileId?: string,
  ) => void;
  onCloseShowModal: () => void;

  // Change Avatar Modal
  isChangeModalOpen: boolean;
  onOpenChangeModal: (avatar?: IAvatar | null, profileId?: string) => void;
  onCloseChangeModal: () => void;

  // Upload Avatar Modal
  isUploadModalOpen: boolean;
  onOpenUploadModal: () => void;
  onCloseUploadModal: () => void;

  // Set Default Avatar Modal
  isSetDefaultModalOpen: boolean;
  onOpenSetDefaultModal: (
    profileId: string,
    defaultAvatar?: IDefaultAvatar,
  ) => void;
  onCloseSetDefaultModal: () => void;
}

export const useAvatarModalStore = create<AvatarModalStore>((set) => ({
  // Shared Data
  avatarUrl: "",
  avatarObj: null,
  username: "",
  userId: "",
  profileId: "",
  defaultAvatarObj: null,

  // Show Avatar Modal
  isShowModalOpen: false,
  onOpenShowModal: (url, name, userId, profileId) =>
    set({
      isShowModalOpen: true,
      avatarUrl: url,
      username: name,
      userId,
      profileId: profileId || "",
    }),
  onCloseShowModal: () => set({ isShowModalOpen: false }),

  // Change Avatar Modal
  isChangeModalOpen: false,
  onOpenChangeModal: (avatar, profileId) =>
    set({
      isShowModalOpen: false, // Digərini bağlayıb bunu açırıq
      isChangeModalOpen: true,
      avatarObj: avatar,
      profileId: profileId || "",
    }),
  onCloseChangeModal: () => set({ isChangeModalOpen: false }),

  // Upload Avatar Modal
  isUploadModalOpen: false,
  onOpenUploadModal: () =>
    set({
      isChangeModalOpen: false, // Digərini bağlayıb bunu açırıq
      isUploadModalOpen: true,
    }),
  onCloseUploadModal: () => set({ isUploadModalOpen: false }),

  // Set Default Avatar Modal
  isSetDefaultModalOpen: false,
  onOpenSetDefaultModal: (profileId, defaultAvatar) =>
    set({
      isChangeModalOpen: false,
      isSetDefaultModalOpen: true,
      defaultAvatarObj: defaultAvatar,
      profileId: profileId || "",
    }),
  onCloseSetDefaultModal: () => set({ isSetDefaultModalOpen: false }),
}));

```

## File: connectfy-client/src/routes/router.tsx
```typescript
import { lazy } from "react";
import {
  InsideProfile,
  RedirectMain,
  RequireAuth,
} from "@/components/routeGuard/RequireAuth";
import { ROUTER } from "@/common/constants/routet";
import { Navigate } from "react-router-dom";
import Loader from "@/components/Loader/Main/Loader.tsx";

// ======================= LAYOUT
const AuthLayout = Loader(lazy(() => import("@/layouts/Auth/AuthLayout")));
const BaseLayout = Loader(lazy(() => import("@/layouts/Base/BaseLayout")));
const SettingsLayout = lazy(() => import("@/layouts/Settings/SettingsLayout"));
const UsersLayout = lazy(() => import("@/layouts/Users/UsersLayout"));

const Messenger = lazy(() => import("@/modules/messenger/ui/Messenger"));

// ======================= AUTH
import authRoutes from "@/modules/auth/router/router";

// ======================= TERMS AND CONDITIONS
import termsAndConditionsRoutes from "@/modules/termsAndConditions/router/router";

// ======================= USERS
import usersRoutes from "@/modules/users/Users/router/router";
import allUsersRoutes from "@/modules/users/AllUsers/router/router";
import userProfileRoutes from "@/modules/users/UserProfile/router/router";
import myFriendsRoutes from "@/modules/users/MyFriends/router/router";
import blocklistRoutes from "@/modules/users/Blocklist/router/router";

// ======================= NOTIFICATIONS
import notificationsRoutes from "@/modules/notifications/router/router";

// ======================= SETTINGS
import settingsRoutes from "@/modules/settings/Settings/router/router";
import generalSettingsRoutes from "@/modules/settings/GeneralSettings/router/router";
import accountSettingsRoutes from "@/modules/settings/AccountSettings/router/router";
import privacySettingsRoutes from "@/modules/settings/PrivacySettings/router/router";
import notificationSettingsRoutes from "@/modules/settings/NotificationSettings/router/router";
import friendRequestsRoutes from "@/modules/users/FriendshipRequests/router/router";

// ======================= PROFILE
import profileRoutes from "@/modules/profile/router/router";

const routes = [
  // ======================= ROOT REDIRECT
  {
    path: "/",
    element: <RedirectMain />,
  },
  // ======================= AUTH
  {
    path: "/",
    element: (
      <InsideProfile>
        <AuthLayout />
      </InsideProfile>
    ),
    children: [
      ...authRoutes,
      {
        path: "*",
        element: (
          <Navigate to={`${ROUTER.AUTH.LOGIN}?method=username`} replace />
        ),
      },
    ],
  },
  // ======================= TERMS AND CONDITIONS
  ...termsAndConditionsRoutes,
  // ======================= MAIN APP
  {
    path: "/",
    element: (
      <RequireAuth>
        <BaseLayout />
      </RequireAuth>
    ),
    children: [
      // ======================= MESSENGER
      {
        path: ROUTER.MESSENGER.MAIN,
        element: <Messenger />,
      },

      // ======================= PROFILE
      ...profileRoutes,
      ...userProfileRoutes,

      // ======================= USERS
      {
        path: "/",
        element: <UsersLayout />,
        children: [
          ...usersRoutes,
          ...allUsersRoutes,
          ...myFriendsRoutes,
          ...friendRequestsRoutes,
          ...blocklistRoutes,
        ],
      },

      // ======================= NOTIFICATIONS
      ...notificationsRoutes,

      // ======================= SETTINGS
      {
        path: "/",
        element: <SettingsLayout />,
        children: [
          // {
          //   index: true,
          //   element: <Settings />,
          // },
          ...settingsRoutes,
          ...generalSettingsRoutes,
          ...accountSettingsRoutes,
          ...privacySettingsRoutes,
          ...notificationSettingsRoutes,
        ],
      },

      // ======================= *
      { path: "*", element: <Navigate to={ROUTER.MESSENGER.MAIN} replace /> },
    ],
  },
];

export default routes;

```

## File: connectfy-client/src/hooks/useContextMenu.tsx
```typescript
import { ContextMenuContext } from "@/context/ContextMenuContext";
import { useContext } from "react";

export const useContextMenu = () => {
  const context = useContext(ContextMenuContext);
  if (!context) {
    throw new Error("useContextMenu must be used within ContextMenuProvider");
  }

  const handleContextMenu = (e: React.MouseEvent, content: React.ReactNode) => {
    e.preventDefault();
    context.openMenu(e.clientX, e.clientY, content);
  };

  return { handleContextMenu, closeMenu: context.closeMenu };
};

```

## File: connectfy-client/src/hooks/useShare.tsx
```typescript
import { useState } from "react";
import WhatsappIcon from "@/assets/icons/WhatsappIcon";
import TelegramIcon from "@/assets/icons/TelegramIcon";
import FacebookIcon from "@/assets/icons/FacebookIcon";
import XIcon from "@/assets/icons/XIcon";

interface ShareOptions {
  url: string;
  text?: string;
  title?: string;
}

export const useShare = ({ url, text }: ShareOptions) => {
  const [isOpen, setIsOpen] = useState(false);

  const shareLinks = [
    {
      label: "Telegram",
      icon: <TelegramIcon className="w-5 h-5" />,
      href: `https://t.me/share/url?url=${encodeURIComponent(url)}&text=${encodeURIComponent(text || "")}`,
    },
    {
      label: "WhatsApp",
      icon: <WhatsappIcon className="w-5 h-5" />,
      href: `https://wa.me/?text=${encodeURIComponent(`${text} ${url}`)}`,
    },
    {
      label: "X (Twitter)",
      icon: <XIcon className="w-5 h-5" />,
      href: `https://twitter.com/intent/tweet?url=${encodeURIComponent(url)}&text=${encodeURIComponent(text || "")}`,
    },
    {
      label: "Facebook",
      icon: <FacebookIcon className="w-5 h-5" />,
      href: `https://www.facebook.com/sharer/sharer.php?u=${encodeURIComponent(url)}`,
      action: "open",
    },
  ];

  const handleOpen = (href: string) => {
    window.open(href, "_blank", "noopener,noreferrer");
    setIsOpen(false);
  };

  return { isOpen, setIsOpen, shareLinks, handleOpen };
};

```

## File: connectfy-client/src/hooks/useBoolean.tsx
```typescript
import { useState } from "react";

export default function useBoolean() {
  const [open, setOpen] = useState<boolean>(false);

  const onToggle = () => {
    setOpen(!open);
  };
  const onClose = () => {
    setOpen(false);
  };
  const onOpen = () => {
    setOpen(true);
  };
  return { onToggle, onClose, open, onOpen };
}

```

## File: connectfy-client/src/hooks/useErrors.tsx
```typescript
import { snack } from "@/common/utils/snackManager";
import { FormikErrors } from "formik";

export const useErrors = () => {
  // ================================
  // Api error messages handler
  // ================================
  const showResponseErrors = (error: any) => {
    if (error.message) {
      if (Array.isArray(error.message)) {
        error.message.forEach((err: string) => {
          snack.error(err, { duration: 5000 });
        });
      } else {
        snack.error(error.message, { duration: 5000 });
      }
    }
  };

  // ================================
  // Formik errors handler
  // ================================
  const extract = (obj: any) => {
    if (!obj) return;

    if (typeof obj === "string") {
      snack.warning(obj, { duration: 5000 });
      return;
    }

    Object.values(obj).forEach((value) => extract(value));
  };

  const showFormikErrors = (errors: FormikErrors<any>) => {
    extract(errors);
  };
  return {
    showResponseErrors,
    showFormikErrors,
  };
};

```

## File: connectfy-client/src/hooks/useIsMobile.tsx
```typescript
import { useEffect, useState } from "react";

export function useIsMobile(breakpoint = 1024) {
  const [isMobile, setIsMobile] = useState(
    () => window.innerWidth < breakpoint,
  );

  useEffect(() => {
    const mq = window.matchMedia(`(max-width: ${breakpoint - 1}px)`);

    const handler = (e: MediaQueryListEvent) => setIsMobile(e.matches);

    mq.addEventListener("change", handler);
    return () => mq.removeEventListener("change", handler);
  }, [breakpoint]);

  return isMobile;
}

```

## File: connectfy-client/src/hooks/useTheme.ts
```typescript
// import { useContext } from "react";
// import { ThemeContext } from "@/context/ThemeContext";

// export const useTheme = () => {
//   const ctx = useContext(ThemeContext);
//   if (!ctx) throw new Error("useTheme must be used within ThemeProvider");
//   return ctx;
// };

```

## File: connectfy-client/src/hooks/useFormDisabled.ts
```typescript
import { useEffect, useState } from "react";
import { FormikProps } from "formik";

type ValidationRule<T = any> = boolean | ((values: T) => boolean);

interface UseFormDisabledConfig<T = any> {
  formik: FormikProps<T>;
  loading?: boolean;
  validationRules: ValidationRule<T>[];
}

const useFormDisabled = <T = any>(
  config: UseFormDisabledConfig<T>
): boolean => {
  const [isDisabled, setIsDisabled] = useState(true);

  useEffect(() => {
    const { formik, loading, validationRules } = config;

    if (loading) {
      setIsDisabled(true);
      return;
    }

    const isValid = validationRules.every((rule) => {
      if (typeof rule === "function") {
        return rule(formik.values);
      }
      return rule;
    });

    setIsDisabled(!isValid);
  }, [config.formik.values, config.loading, config.validationRules]);

  return isDisabled;
};

export default useFormDisabled;

```

## File: connectfy-client/src/hooks/useBeforeUnload.ts
```typescript
import { useEffect } from "react";

export function useBeforeUnload(
  when: boolean,
  message = "You have unsaved changes."
) {
  useEffect(() => {
    if (!when) return;

    const handler = (e: BeforeUnloadEvent) => {
      e.preventDefault();
      e.returnValue = message;
      return message;
    };

    window.addEventListener("beforeunload", handler);
    return () => window.removeEventListener("beforeunload", handler);
  }, [when, message]);
}

```

## File: connectfy-client/src/hooks/useAppNavigation.tsx
```typescript
import { useTransition, useCallback } from "react";
import { useNavigate, NavigateOptions, To } from "react-router-dom";

export const useAppNavigation = () => {
  const [isPending, startTransition] = useTransition();
  const navigate = useNavigate();

  const smartNavigate = useCallback(
    (to: To | number, options?: NavigateOptions) => {
      startTransition(() => {
        if (typeof to === "number") {
          navigate(to);
        } else {
          navigate(to, options);
        }
      });
    },
    [navigate],
  );

  return {
    navigate: smartNavigate,
    isPending,
  };
};

```

## File: connectfy-client/src/hooks/useBlocker.tsx
```typescript
import { useCallback, useEffect, useRef, useState } from "react";
import { history } from "@/common/helpers/history";

interface PendingTransition {
  retry?: () => void;
  location?: any;
  action?: string;
  message?: string;
};

export function useBlocker(when: boolean) {
  const [pending, setPending] = useState<PendingTransition | null>(null);
  const unblockRef = useRef<() => void | null>(null);

  useEffect(() => {
    if (!when) return;

    const unblock = history.block((tx: any) => {
      setPending({
        retry: typeof tx.retry === "function" ? tx.retry : undefined,
        location: tx.location ?? tx,
        action: tx.action,
      });
    });

    unblockRef.current = unblock;
    return () => {
      if (unblockRef.current) unblockRef.current();
      unblockRef.current = null;
    };
  }, [when]);

  const confirm = useCallback(() => {
    if (unblockRef.current) {
      unblockRef.current();
      unblockRef.current = null;
    }

    if (!pending) return;

    if (pending.retry) {
      pending.retry();
    } else if (pending.location) {
      const loc = pending.location;
      const to = typeof loc === "string" ? loc : loc.pathname ?? loc;
      history.push(to);
    }
    setPending(null);
  }, [pending]);

  const cancel = useCallback(() => {
    setPending(null);
  }, []);

  return { pending, confirm, cancel };
}

```

## File: connectfy-client/src/layouts/Auth/AuthLayout.tsx
```typescript
import { FC, memo, useEffect, useState } from "react";
import { Outlet, useLocation } from "react-router-dom";
import AuthSidebar from "@/components/Sidebar/Auth/AuthSidebar";
import MainIcon from "@/assets/icons/MainIcon";
import AuthFooter from "@/modules/auth/ui/components/Footer/AuthFooter/AuthFooter";
import { ROUTER } from "@/common/constants/routet";
import { useIsMobile } from "@/hooks/useIsMobile";

const AuthLayout: FC = () => {
  const location = useLocation();

  const [isSignupPage, setIsSignupPage] = useState<boolean>(false);
  const isMobile = useIsMobile();

  useEffect(() => {
    if (location.pathname === ROUTER.AUTH.SIGNUP) {
      setIsSignupPage(true);
    } else {
      setIsSignupPage(false);
    }
  }, [location]);

  return (
    <div
      className="min-h-screen w-full font-sans antialiased text-(--text-(--primary-color)) bg-(--auth-main-bg)"
      style={{
        fontFamily: "'Inter', system-ui, -apple-system, sans-serif",
      }}
    >
      <div className="flex min-h-screen lg:h-screen w-full flex-col lg:flex-row">
        <AuthSidebar />

        <div
          className={`flex flex-1 flex-col items-center justify-center px-6 ${isSignupPage && !isMobile ? "py-5" : "py-12"} lg:px-16 relative overflow-y-auto`}
        >
          {/* Mobile logo */}
          <div className="flex items-center gap-2 mb-10 lg:hidden">
            <MainIcon
              className="size-10 rounded-xl flex items-center justify-center"
              styles={{
                backgroundColor: "var(--auth-sidebar-glow)",
                color: "#fff",
                width: 45,
                height: 45,
                padding: 8,
              }}
            />
            <h2 className="text-3xl font-extrabold tracking-tighter">
              Connectfy
            </h2>
          </div>

          <div className="w-full max-w-[500px] space-y-10 min-h-0 contain-layout">
            <Outlet />

            <AuthFooter />
          </div>
        </div>
      </div>
    </div>
  );
};

export default memo(AuthLayout);

```

## File: connectfy-client/src/layouts/Users/UsersLayout.tsx
```typescript
import { FC, memo, ReactNode } from "react";
import "./usersLayout.style.css";
import { Outlet, useLocation, matchPath } from "react-router-dom";
import UniqueSidebar from "@/components/Sidebar/UniqueSidebar/UniqueSidebar";
import { useTranslation } from "react-i18next";
import { ROUTER } from "@/common/constants/routet";
import {
  Users,
  UserSearch,
  UsersIcon,
  UserPlus,
  UserRoundX,
} from "lucide-react";
import { useAppNavigation } from "@/hooks/useAppNavigation";
import { useIsMobile } from "@/hooks/useIsMobile";

interface Props {
  children?: ReactNode;
}

const UsersLayout: FC<Props> = ({ children }) => {
  const location = useLocation();
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();

  const TITLE = {
    name: t("common.users"),
    icon: Users,
  };

  const SUBJECTS = [
    {
      name: t("common.find_user"),
      path: ROUTER.USERS.SEARCH,
      icon: UserSearch,
      key: "search",
      badge: null,
      onClick: () => navigate(ROUTER.USERS.SEARCH),
    },
    {
      name: t("common.friends"),
      path: ROUTER.USERS.FRIENDS,
      icon: UsersIcon,
      key: "friends",
      badge: null,
      onClick: () => navigate(ROUTER.USERS.FRIENDS),
    },
    {
      name: t("common.friendship_requests"),
      path: ROUTER.USERS.REQUESTS,
      icon: UserPlus,
      key: "requests",
      badge: null,
      onClick: () => navigate(ROUTER.USERS.REQUESTS),
    },
    {
      name: t("common.blocked_users"),
      path: ROUTER.USERS.BLOCKLIST,
      icon: UserRoundX,
      key: "blocklist",
      badge: null,
      onClick: () => navigate(ROUTER.USERS.BLOCKLIST),
    },
  ];

  const isMobile = useIsMobile();

  const isIndex = Boolean(
    matchPath({ path: ROUTER.USERS.MAIN, end: true }, location.pathname),
  );

  if (!isMobile) {
    return (
      <section id="users-layout" className="users-layout desktop">
        <div className="users-layout__sidebar">
          <UniqueSidebar title={TITLE} subjects={SUBJECTS} />
        </div>
        <div className="users-layout__content">{children || <Outlet />}</div>
      </section>
    );
  }

  if (isMobile && isIndex) {
    return (
      <section id="users-layout" className="users-layout mobile sidebar-only">
        <div className="users-layout__sidebar-mobile">
          <UniqueSidebar title={TITLE} subjects={SUBJECTS} />
        </div>
      </section>
    );
  }

  return (
    <section id="users-layout" className="users-layout mobile content-only">
      <div className="users-layout__content">{children || <Outlet />}</div>
    </section>
  );
};

export default memo(UsersLayout);

```

## File: connectfy-client/src/layouts/Settings/SettingsLayout.tsx
```typescript
import { FC, memo, ReactNode } from "react";
import "./settingsLayout.style.css";
import { Outlet, useLocation, matchPath } from "react-router-dom";
import UniqueSidebar from "@/components/Sidebar/UniqueSidebar/UniqueSidebar";
import { useTranslation } from "react-i18next";
import { ROUTER } from "@/common/constants/routet";
import {
  Settings,
  KeyRound,
  Paintbrush,
  UserCog,
  BellRing,
  Keyboard,
} from "lucide-react";
import { useAppNavigation } from "@/hooks/useAppNavigation";
import { useIsMobile } from "@/hooks/useIsMobile";

interface Props {
  children?: ReactNode;
}

const SettingsLayout: FC<Props> = ({ children }) => {
  const location = useLocation();
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();

  const TITLE = {
    name: t("common.settings"),
    icon: Settings,
  };

  const SUBJECTS = [
    {
      name: t("common.general_settings"),
      path: ROUTER.SETTINGS.GENERAL,
      icon: Settings,
      key: "general",
      badge: null,
      onClick: () => navigate(ROUTER.SETTINGS.GENERAL),
    },
    {
      name: t("common.account_settings"),
      path: ROUTER.SETTINGS.ACCOUNT,
      icon: UserCog,
      key: "account",
      badge: null,
      onClick: () => navigate(ROUTER.SETTINGS.ACCOUNT),
    },
    {
      name: t("common.privacy_settings"),
      path: ROUTER.SETTINGS.PRIVACY,
      icon: KeyRound,
      key: "privacy",
      badge: null,
      onClick: () => navigate(ROUTER.SETTINGS.PRIVACY),
    },
    {
      name: t("common.notification_settings"),
      path: ROUTER.SETTINGS.NOTIFICATION,
      icon: BellRing,
      key: "notification",
      badge: null,
      onClick: () => navigate(ROUTER.SETTINGS.NOTIFICATION),
    },
    {
      name: t("common.change_background"),
      path: ROUTER.SETTINGS.BACKGROUND,
      icon: Paintbrush,
      key: "background",
      badge: null,
      onClick: () => navigate(ROUTER.SETTINGS.BACKGROUND),
    },
    {
      name: t("common.keyboard_shortcuts"),
      path: ROUTER.SETTINGS.SHORTCUT,
      icon: Keyboard,
      key: "keyboard",
      badge: null,
      onClick: () => navigate(ROUTER.SETTINGS.SHORTCUT),
    },
  ];

  const isMobile = useIsMobile();

  const isIndex = Boolean(
    matchPath({ path: ROUTER.SETTINGS.MAIN, end: true }, location.pathname),
  );

  if (!isMobile) {
    return (
      <section id="settings-layout" className="settings-layout desktop">
        <div className="settings-layout__sidebar">
          <UniqueSidebar title={TITLE} subjects={SUBJECTS} />
        </div>
        <div className="settings-layout__content">{children || <Outlet />}</div>
      </section>
    );
  }

  if (isMobile && isIndex) {
    return (
      <section
        id="settings-layout"
        className="settings-layout mobile sidebar-only"
      >
        <div className="settings-layout__sidebar-mobile">
          <UniqueSidebar title={TITLE} subjects={SUBJECTS} />
        </div>
      </section>
    );
  }

  return (
    <section
      id="settings-layout"
      className="settings-layout mobile content-only"
    >
      <div className="settings-layout__content">{children || <Outlet />}</div>
    </section>
  );
};

export default memo(SettingsLayout);

```

## File: connectfy-client/src/layouts/Base/BaseLayout.tsx
```typescript
import "./baseLayout.style.css";
import DesktopSidebar from "@/components/Sidebar/Desktop/DesktopSidebar";
import MobileSidebar from "@/components/Sidebar/Mobile/MobileSidebar";
import { useIsMobile } from "@/hooks/useIsMobile";
import { FC, memo, ReactNode } from "react";
import { Outlet } from "react-router-dom";

interface Props {
  children?: ReactNode;
}

const BaseLayout: FC<Props> = ({ children }) => {
  const isMobile = useIsMobile();

  return (
    <section
      id="layout"
      className={isMobile ? "mobile-layout" : "desktop-layout"}
    >
      {/* Desktop Sidebar */}
      {!isMobile && (
        <div className="layout-sidebar-box">
          <DesktopSidebar />
        </div>
      )}

      {/* Content */}
      <div className="layout-content-box">{children || <Outlet />}</div>

      {/* Mobile Sidebar */}
      {isMobile && <MobileSidebar />}
    </section>
  );
};

export default memo(BaseLayout);

```

## File: connectfy-client/src/modules/profile/hooks/useUpdateAvatar.tsx
```typescript
import { useState } from "react";
import axios from "axios";
import api from "@/common/api/axios";
import { useUpdateAvatarMutation } from "../api/api";
import { useErrors } from "@/hooks/useErrors";
import { ProfilePhotoUpdateAction } from "@/common/enums/enums";
import { API_ENDPOINTS } from "@/common/constants/apiEndpoints";
import { snack } from "@/common/utils/snackManager";
import {
  ALLOWED_TYPES,
  MAX_SIZE_BYTES,
  MAX_SIZE_MB,
} from "../constants/constants";
import { useTranslation } from "react-i18next";

export const useUpdateAvatar = (currentAvatarUrl?: string) => {
  const { t } = useTranslation();
  const [updateAvatar, { isLoading }] = useUpdateAvatarMutation();
  const { showResponseErrors } = useErrors();
  const [previewUrl, setPreviewUrl] = useState<string | null>(null);

  const validateFile = (file: File): string | null => {
    if (!ALLOWED_TYPES.includes(file.type)) {
      const formats = ALLOWED_TYPES.map((type) =>
        type.split("/")[1].toUpperCase(),
      ).join(", ");

      return t("error_messages.invalid_file_type", {
        formats,
      });
    }
    if (file.size > MAX_SIZE_BYTES) {
      return t("error_messages.file_too_large", {
        size: MAX_SIZE_MB,
      });
    }
    return null;
  };

  const handleAvatarUpload = async ({
    _id,
    file,
    action,
  }: {
    _id: string;
    file?: File;
    action: ProfilePhotoUpdateAction;
  }) => {
    let objectUrl: string | null = null; // Scope-u yuxarı qaldırdıq

    try {
      if (action === ProfilePhotoUpdateAction.Update && file) {
        const validationError = validateFile(file);
        if (validationError) {
          snack.error(validationError);
          return;
        }

        // 1. Optimistic UI Update
        objectUrl = URL.createObjectURL(file);
        setPreviewUrl(objectUrl);

        // 2. Presigned URL
        const { data: presignedData } = await api.post(
          API_ENDPOINTS.FILES.PRESIGNED_UPLOAD,
          {
            filename: file.name,
            moduleName: "profile",
            contentType: file.type,
          },
        );

        // 3. Direct S3 Upload
        await axios.put(presignedData.uploadUrl, file, {
          headers: { "Content-Type": file.type },
        });

        const payload = {
          _id,
          action,
          avatar: {
            key: presignedData.fileKey,
            url: presignedData.fileUrl,
            isCustom: true,
          },
        };

        // 4. Commit to Backend
        await updateAvatar(payload).unwrap();
      } else {
        // Default və ya Remove əməliyyatları
        await updateAvatar({ _id, action, avatar: null }).unwrap();
      }
    } catch (error) {
      console.error("❌ Process failed:", error);
      showResponseErrors(error);
      setPreviewUrl(null); // Rollback
    } finally {
      if (objectUrl) URL.revokeObjectURL(objectUrl);
    }
  };

  const displayAvatar = previewUrl || currentAvatarUrl;
  return { handleAvatarUpload, displayAvatar, validateFile, isLoading };
};

```

## File: connectfy-client/src/modules/profile/hooks/useProfile.tsx
```typescript
import { useGetAccountQuery } from "@/modules/profile/api/api.ts";
import { useAuthStore } from "@/store/zustand/useAuthStore";

export function useProfile() {
  const { access_token } = useAuthStore();

  const result = useGetAccountQuery(undefined, {
    skip: !access_token,
  });

  return {
    profile: result.data,
    ...result,
  };
}

```

## File: connectfy-client/src/modules/profile/constants/validation.ts
```typescript
import { checkEmptyString, checkUrl } from "@/common/utils/checkValues";
import { IAddSocialLink, IEditProfile, IEditSocialLink } from "../types/types";
import { TFunction } from "i18next";
import { GENDER } from "@/common/enums/enums";

export const validateEditProfile = (values: IEditProfile, t: TFunction) => {
  const { firstName, lastName, gender, birthdayDate } = values;
  const errors: Record<string, any> = {};

  if (!firstName || !checkEmptyString(firstName)) {
    errors.firstName = t("error_messages.first_name_is_required");
  }

  if (!lastName || !checkEmptyString(lastName)) {
    errors.lastName = t("error_messages.last_name_is_required");
  }

  if (!gender || !Object.values(GENDER).includes(gender)) {
    errors.gender = t("error_messages.gender_is_required");
  }

  if (!birthdayDate) {
    errors.birthdayDate = t("error_messages.birthday_is_required");
  }

  return errors;
};

export const validateSocialLink = (
  values: IEditSocialLink | IAddSocialLink,
  t: TFunction,
) => {
  const errors: Record<string, string> = {};
  if (!values.name || !checkEmptyString(values.name)) {
    errors.name = t("error_messages.name_is_required");
  }
  if (!values.url || !checkEmptyString(values.url)) {
    errors.url = t("error_messages.url_is_required");
  }
  if (values.url && !checkUrl(values.url)) {
    errors.url = t("error_messages.url_invalid");
  }
  if (!values.platform || !checkEmptyString(values.platform)) {
    errors.platform = t("error_messages.platform_required");
  }
  return errors;
};

```

## File: connectfy-client/src/modules/profile/constants/constants.ts
```typescript
export const ALLOWED_TYPES = [
  "image/jpeg",
  "image/png",
  "image/webp",
  "image/gif",
];
export const MAX_SIZE_MB = 50;
export const MAX_SIZE_BYTES = MAX_SIZE_MB * 1024 * 1024;

```

## File: connectfy-client/src/modules/profile/ui/Profile.tsx
```typescript
import {
  FC,
  Fragment,
  memo,
  useCallback,
  useEffect,
  useState,
  useTransition,
} from "react";
import ProfileHeader from "./components/ProfileHeader/ProfileHeader";
import MainCard from "./components/MainCard/MainCard";
import PersonalInformation from "./components/PersonalInformation/PersonalInformation";
import Bio from "./components/Bio/Bio";
import SocialLinks from "./components/SocialLinks/SocialLinks";
import { useUser } from "@/context/UserContext";
import { useProfile } from "../hooks/useProfile";
import { usePrivacySettings } from "@/modules/settings/PrivacySettings/hooks/usePrivacySettings";
import MainCardSkeleton from "@/components/Skeleton/profile/MainCardSkeleton";
import PersonalInformationSkeleton from "@/components/Skeleton/profile/PersonalInformationSkeleton";
import BioSkeleton from "@/components/Skeleton/profile/BioSkeleton";
import OnlineFriendsSidebar from "./components/OnlineFriendsSidebar";

const Profile: FC = () => {
  const { user, isLoading: userLoading, hasPhoneNumber } = useUser();
  const { profile, isLoading: profileLoading } = useProfile();
  const { privacySettings, isLoading: privacyLoading } = usePrivacySettings();

  const [_, startTransition] = useTransition();
  const [showContent, setShowContent] = useState(false);
  const [isFriendsSidebarOpen, setIsFriendsSidebarOpen] = useState(false);

  const isDataReady = !userLoading && !profileLoading && !privacyLoading;

  const toggleFriendsSidebar = useCallback(() => {
    setIsFriendsSidebarOpen((prev) => !prev);
  }, []);

  useEffect(() => {
    if (isDataReady) {
      // Browser click eventlərini bu render-dən önə keçirə bilər
      startTransition(() => {
        setShowContent(true);
      });
    } else {
      setShowContent(false);
    }
  }, [isDataReady]);

  return (
    // Əsas flex konteyner (bütün ekranı tutur və scroll gizlədilir, scroll daxili div-lərdə olacaq)
    <div className="flex w-full h-screen overflow-hidden bg-(--bg-color) font-sans">
      {/* Sol tərəf: Əsas Profil Məlumatları */}
      <div className="flex-1 h-full overflow-x-hidden overflow-y-auto scroll-smooth">
        <ProfileHeader onToggleFriends={toggleFriendsSidebar} />

        <main className="mx-auto max-w-[900px] pt-8 px-6 pb-[60px] animate-slide-up">
          {!showContent ? (
            <Fragment>
              <MainCardSkeleton />
              <PersonalInformationSkeleton />
              <BioSkeleton />
            </Fragment>
          ) : (
            <Fragment>
              <MainCard user={user} profile={profile} />
              <PersonalInformation
                user={user}
                profile={profile}
                privacySettings={privacySettings}
                hasPhoneNumber={!!hasPhoneNumber}
              />
              <Bio profile={profile} privacySettings={privacySettings} />
            </Fragment>
          )}

          <SocialLinks userId={user?._id} />
        </main>
      </div>

      {/* Sağ tərəf: Səhifəni bölən Sidebar */}
      <OnlineFriendsSidebar isOpen={isFriendsSidebarOpen} userId={user?._id} />
    </div>
  );
};

export default memo(Profile);

```

## File: connectfy-client/src/modules/profile/ui/components/OnlineFriendsSidebar.tsx
```typescript
import { FC, memo, useState } from "react";
import { motion, AnimatePresence } from "framer-motion";
import { MessageSquare, UserPlus, Clock } from "lucide-react";
import { useTranslation } from "react-i18next";
import Button from "@/components/ui/CustomButton/Button/Button";
import {
  useFindOnlineFriendsQuery,
  useFindSuggestionsQuery,
} from "@/modules/users/MyFriends/api/api";
import { useAppNavigation } from "@/hooks/useAppNavigation";
import { ROUTER } from "@/common/constants/routet";
import NoProfilePhotoIcon from "@/assets/icons/NoProfilePhotoIcon";
import { useFriendship } from "@/modules/users/MyFriends/hooks/useFriendship";

interface OnlineFriendsSidebarProps {
  isOpen: boolean;
  userId?: string;
}

const OnlineFriendsSidebar: FC<OnlineFriendsSidebarProps> = ({
  isOpen,
  userId,
}) => {
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();

  const { data: onlineData, isLoading: isOnlineLoading } =
    useFindOnlineFriendsQuery(
      { userId: userId!, skip: 1, limit: 20 },
      { skip: !isOpen || !userId },
    );

  const { data: suggestionsData, isLoading: isSuggestionsLoading } =
    useFindSuggestionsQuery(
      { userId: userId!, skip: 1, limit: 20 },
      { skip: !isOpen || !userId },
    );

  const [pendingRequests, setPendingRequests] = useState<Record<string, string>>({});

  const { handleSendFriendRequest, handleCancelFriendRequest, isActionLoading } = useFriendship();

  const handleAddFriend = async (friendId: string, e: React.MouseEvent) => {
    e.stopPropagation();
    const res = await handleSendFriendRequest({ friendId });
    if (res && res._id) {
      setPendingRequests(prev => ({ ...prev, [friendId]: res._id }));
    }
  };

  const handleCancelFriend = async (friendId: string, friendshipId: string, e: React.MouseEvent) => {
    e.stopPropagation();
    await handleCancelFriendRequest({ friendshipId, userId: friendId });
    setPendingRequests(prev => {
      const newReqs = { ...prev };
      delete newReqs[friendId];
      return newReqs;
    });
  };

  const handleNavigate = (id: string, e: React.MouseEvent) => {
    e.stopPropagation();
    navigate(`${ROUTER.USERS.PROFILE}/${id}`);
  };

  const onlineFriends = Array.from(
    new Map((onlineData?.data || []).map((f: any) => [f._id, f])).values(),
  );
  const onlineCount = onlineData?.onlineCount || 0;
  const suggestions = Array.from(
    new Map(
      (suggestionsData?.data || []).map((s: any) => [s.userId, s]),
    ).values(),
  );

  return (
    <AnimatePresence initial={false}>
      {isOpen && (
        <motion.aside
          initial={{ width: 0, opacity: 0 }}
          animate={{ width: 400, opacity: 1 }}
          exit={{ width: 0, opacity: 0 }}
          transition={{ type: "spring", bounce: 0, duration: 0.4 }}
          className="h-full bg-(--bg-color) border-l border-(--input-border) shrink-0 overflow-hidden"
        >
          {/* Məzmunun animasiya vaxtı qırılmaması üçün daxili sabit genişlik */}
          <div className="w-[400px] h-full flex flex-col overflow-y-auto custom-scrollbar">
            {/* Header */}
            <div className="flex items-center justify-between p-6 pb-4">
              <h2 className="text-[22px] font-bold text-(--text-color)">
                {t("common.online_friends", "Online Friends")}
              </h2>
              <span className="px-3 py-1 text-xs font-bold rounded-full bg-(--icon-green-bg) text-(--icon-green-text)">
                {onlineCount}
              </span>
            </div>

            {/* Online Friends List */}
            <div className="flex flex-col px-4 pb-6">
              {onlineFriends.map((friend: any) => (
                <div
                  key={friend._id}
                  onClick={(e) => handleNavigate(friend._id, e)}
                  className="flex items-center justify-between p-2.5 rounded-xl hover:bg-(--disabled-bg) transition-colors cursor-pointer group"
                >
                  <div className="flex items-center gap-3">
                    <div className="relative">
                      {friend.avatar ? (
                        <img
                          src={friend.avatar}
                          alt={friend.firstName}
                          className="w-[46px] h-[46px] rounded-full object-cover"
                        />
                      ) : (
                        <div className="flex items-center justify-center w-[46px] h-[46px] rounded-full bg-(--active-bg-2)">
                          <NoProfilePhotoIcon />
                        </div>
                      )}
                      {/* Yaşıl status nöqtəsi */}
                      <span className="absolute bottom-0 right-0 w-3.5 h-3.5 bg-(--primary-color) border-[2.5px] border-(--bg-color) rounded-full"></span>
                    </div>
                    <div className="flex flex-col">
                      <span className="text-[15px] font-bold text-(--text-color)">
                        {friend.fullName || `${friend.firstName} ${friend.lastName}`}
                      </span>
                      <span className="text-[13px] font-medium text-(--muted-color)">
                        @{friend.username}
                      </span>
                    </div>
                  </div>
                  <Button
                    className="text-(--text-disabled) group-hover:text-(--primary-color) transition-colors p-1"
                    onClick={(e: any) => e.stopPropagation()}
                  >
                    <MessageSquare size={22} strokeWidth={1.5} />
                  </Button>
                </div>
              ))}
              {isOnlineLoading && (
                <p className="text-sm text-center text-(--muted-color)">
                  {t("common.loading", "Loading")}...
                </p>
              )}
              {!isOnlineLoading && onlineFriends.length === 0 && (
                <p className="text-sm text-center text-(--muted-color)">
                  {t("common.no_online_friends", "No friends online")}
                </p>
              )}
            </div>

            {/* Divider */}
            <div className="px-6 py-2">
              <div className="w-full h-px bg-(--input-border)"></div>
            </div>

            {/* Suggestions Section */}
            <div className="flex flex-col px-4 py-4">
              <h3 className="px-3 pb-4 text-[12px] font-bold tracking-widest text-(--muted-color) uppercase">
                {t("common.suggestions", "Suggestions")}
              </h3>

              <div className="flex flex-col gap-1">
                {suggestions.map((suggestion: any) => (
                  <div
                    key={suggestion.userId}
                    onClick={(e) => handleNavigate(suggestion.userId, e)}
                    className="flex items-center justify-between p-2.5 rounded-xl hover:bg-(--disabled-bg) transition-colors cursor-pointer group"
                  >
                    <div className="flex items-center gap-3">
                      {suggestion.avatar ? (
                        <img
                          src={suggestion.avatar}
                          alt={suggestion.firstName}
                          className="w-[46px] h-[46px] rounded-full object-cover"
                        />
                      ) : (
                        <div className="flex items-center justify-center w-[46px] h-[46px] rounded-xl bg-(--active-bg-2)">
                          <NoProfilePhotoIcon />
                        </div>
                      )}
                      <div className="flex flex-col">
                        <span className="text-[15px] font-bold text-(--text-color)">
                          {suggestion.fullName || `${suggestion.firstName} ${suggestion.lastName}`}
                        </span>
                        <span className="text-[12px] font-medium text-(--muted-color)">
                          {t("common.mutual_friends", "Mutual friends")}:{" "}
                          {suggestion.mutualCount}
                        </span>
                      </div>
                    </div>
                    {pendingRequests[suggestion.userId] ? (
                      <Button
                        onClick={(e: any) =>
                          handleCancelFriend(suggestion.userId, pendingRequests[suggestion.userId], e)
                        }
                        disabled={isActionLoading}
                        className="text-(--text-color) p-2 rounded-lg opacity-70 border border-(--active-bg) transition-colors"
                      >
                        <Clock size={20} strokeWidth={2} />
                      </Button>
                    ) : (
                      <Button
                        onClick={(e: any) =>
                          handleAddFriend(suggestion.userId, e)
                        }
                        disabled={isActionLoading}
                        className="text-(--primary-color) p-2 rounded-lg hover:bg-(--active-bg-2) transition-colors"
                      >
                        <UserPlus size={20} strokeWidth={2} />
                      </Button>
                    )}
                  </div>
                ))}
                {isSuggestionsLoading && (
                  <p className="text-sm text-center text-(--muted-color)">
                    {t("common.loading", "Loading")}...
                  </p>
                )}
                {!isSuggestionsLoading && suggestions.length === 0 && (
                  <p className="text-sm text-center text-(--muted-color)">
                    {t("common.no_suggestions", "No suggestions")}
                  </p>
                )}
              </div>
            </div>

            {/* Footer Action */}
            <div className="px-6 pb-8 mt-auto pt-4">
              <Button className="w-full py-3.5 text-[15px] font-bold text-(--muted-color) border-2 border-dashed border-(--input-border) rounded-[14px] hover:bg-(--disabled-bg) hover:text-(--text-color) transition-all">
                {t("common.more", "More")}
              </Button>
            </div>
          </div>
        </motion.aside>
      )}
    </AnimatePresence>
  );
};

export default memo(OnlineFriendsSidebar);

```

## File: connectfy-client/src/modules/profile/ui/components/Bio/Bio.tsx
```typescript
import { FC, Fragment } from "react";
import { UserPen, MinusIcon } from "lucide-react";
import { useTranslation } from "react-i18next";
import { PRIVACY_SETTINGS_CHOICE } from "@/common/enums/enums";
import Button from "@/components/ui/CustomButton/Button/Button";
import { PrivacyIcon } from "../PrivacyIcon/PrivacyIcon";
import useBoolean from "@/hooks/useBoolean";
import BioModal from "../Modal/BioModal/BioModal";
import { IAccount } from "@/modules/profile/types/types";
import { IPrivacySettings } from "@/modules/settings/PrivacySettings/types/types";

interface IProps {
  profile: IAccount | undefined;
  privacySettings: IPrivacySettings | undefined;
}

const Bio: FC<IProps> = ({ profile, privacySettings }) => {
  const { t } = useTranslation();
  const { open, onOpen, onClose } = useBoolean();

  return (
    <Fragment>
      <section
        className="p-8 mb-8 transition-all duration-300 bg-(--auth-main-bg) rounded-[20px] border border-(--auth-glass-border) shadow-(--card-shadow)"
        aria-labelledby="bio-heading"
      >
        {/* Header Hissəsi */}
        <div className="flex flex-row items-center justify-between gap-3 mb-6 md:mb-8">
          {/* Sol tərəf: İkon və Başlıq */}
          <div className="flex items-center gap-2 md:gap-3">
            <UserPen className="w-6 h-6 md:w-7 md:h-7 text-(--primary-color)" />
            <h2
              id="bio-heading"
              className="m-0 text-lg font-bold md:text-2xl text-(--text-color)"
            >
              {t("common.bio")}
            </h2>
          </div>

          {/* Sağ tərəf: Məxfilik və Redaktə Düyməsi */}
          <div className="flex items-center gap-2 md:gap-3">
            <PrivacyIcon
              privacy={privacySettings?.bio || PRIVACY_SETTINGS_CHOICE.EVERYONE}
              fieldName="bio"
            />

            {/* Böyük ekranlar üçün (Text ilə) */}
            <Button
              className="hidden lg:flex bg-(--btn-edit-bg)! text-(--btn-edit-text)! font-semibold px-4 py-2 rounded-lg items-center gap-2 transition-all hover:opacity-80 border-none text-sm whitespace-nowrap"
              title={t("common.edit")}
              onClick={onOpen}
            />

            {/* Mobil ekranlar üçün (İkon ilə) */}
            <Button
              className="flex lg:hidden bg-(--btn-edit-bg)! text-(--btn-edit-text)! font-semibold p-3 rounded-lg items-center justify-center transition-all hover:opacity-80 border-none"
              icon={<UserPen size={16} />}
              onClick={onOpen}
            />
          </div>
        </div>

        {/* Bio Mətni */}

        <div className="p-6 rounded-xl bg-(--info-card-bg) border border-(--info-card-border) transition-all duration-200 hover:bg-(--active-bg-2)">
          {profile?.bio ? (
            <div
              className="text-[15px] leading-[1.7] text-(--text-color) prose prose-sm max-w-none [&_strong]:font-bold [&_em]:italic [&_ul]:list-disc [&_ul]:pl-5 [&_ol]:list-decimal [&_ol]:pl-5 [&_s]:line-through"
              dangerouslySetInnerHTML={{ __html: profile.bio }}
            />
          ) : (
            <div className="flex items-center gap-2 text-(--muted-color) italic py-2">
              <MinusIcon size={18} />
              <span>{t("common.no_bio_info")}</span>
            </div>
          )}
        </div>
      </section>

      {open && <BioModal open={open} onClose={onClose} />}
    </Fragment>
  );
};

export default Bio;

```

## File: connectfy-client/src/modules/profile/ui/components/Modal/BioModal/BioModal.tsx
```typescript
import { FC } from "react";
import { useTranslation } from "react-i18next";
import { useFormik } from "formik";
import { UserPen } from "lucide-react";
import Modal from "@/components/Modal";
import Button from "@/components/ui/CustomButton/Button/Button";
import { useErrors } from "@/hooks/useErrors";
import useFormDisabled from "@/hooks/useFormDisabled";
import { useUpdateProfileMutation } from "@/modules/profile/api/api";
import { useProfile } from "@/modules/profile/hooks/useProfile";
import { IEditProfile } from "@/modules/profile/types/types";
import { getChangedData } from "@/common/utils/getDirtyValues";
import { snack } from "@/common/utils/snackManager";
import Textarea from "@/components/ui/CustomTextArea/TextArea/Textarea";

interface IProps {
  open: boolean;
  onClose: () => void;
}

const BioModal: FC<IProps> = ({ open, onClose }) => {
  const { t } = useTranslation();
  const { profile } = useProfile();
  const { showResponseErrors, showFormikErrors } = useErrors();
  const [update, { isLoading }] = useUpdateProfileMutation();

  const initialState: IEditProfile = {
    _id: profile?._id ?? "",
    bio: profile?.bio,
  };

  const formik = useFormik({
    initialValues: initialState,
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    onSubmit: async (values) => {
      try {
        values.bio = values.bio?.trim() || null;
        const changedData = getChangedData<IEditProfile>(profile!, values);

        const finalData = {
          _id: profile?._id as string,
          ...changedData,
        };

        await update(finalData).unwrap();
        snack.success(t("user_messages.information_updated"));
        handleClose();
      } catch (error) {
        showResponseErrors(error);
      }
    },
  });

  const isDisabled = useFormDisabled<IEditProfile>({
    formik,
    loading: isLoading,
    validationRules: [formik.dirty],
  });

  const handleClose = () => {
    formik.resetForm();
    onClose();
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    const errors = await formik.validateForm();
    if (Object.keys(errors).length > 0) {
      showFormikErrors(errors);
      return;
    }
    formik.handleSubmit(e as any);
  };

  return (
    <Modal open={open} onClose={handleClose}>
      <div className="w-full max-w-[550px] bg-(--auth-main-bg) rounded-2xl shadow-(--card-shadow) overflow-hidden animate-slide-up duration-600">
        {/* Modal Header */}
        <div className="p-6 border-b border-(--auth-glass-border)">
          <h2 className="text-xl font-bold text-(--text-primary) flex items-center gap-2">
            <UserPen size={22} className="text-(--primary-color)" />
            {t("common.bio")}
          </h2>
          <p className="text-sm text-(--muted-color) mt-1">
            {t("common.edit_bio_description")}
          </p>
        </div>

        {/* Modal Body */}
        <form onSubmit={handleSubmit} className="p-6">
          <div className="flex flex-col gap-6">
            <Textarea
              title={t("common.bio")}
              maxLength={500}
              showCharCount
              value={formik.values.bio ?? ""}
              onChange={(e) => formik.setFieldValue("bio", e.target.value)}
              style={{ resize: "none" }}
            />
          </div>

          {/* Action Buttons */}
          <div className="flex items-center gap-3 mt-8">
            <Button
              type="button"
              className="flex-1 h-12 rounded-xl text-sm font-bold bg-black/5 dark:bg-white/5 text-(--text-primary) hover:bg-black/10 dark:hover:bg-white/10 transition-all"
              onClick={handleClose}
              disabled={isLoading}
              title={t("common.cancel")}
            />
            <Button
              type="submit"
              className="flex-1 h-12 rounded-xl text-sm font-bold bg-linear-to-r from-(--primary-color) to-(--third-color) text-white shadow-lg shadow-(--primary-color)/20 transition-all duration-600"
              disabled={isDisabled}
              isLoading={isLoading}
              title={t("common.save_changes")}
            />
          </div>
        </form>
      </div>
    </Modal>
  );
};

export default BioModal;

```

## File: connectfy-client/src/modules/profile/ui/components/Modal/PrivacyIconModal/PrivacyIconModal.tsx
```typescript
import { FC, useState } from "react";
import Modal from "@/components/Modal";
import { Globe, Users, Lock, Shield, Check, X } from "lucide-react";
import { PRIVACY_SETTINGS_CHOICE } from "@/common/enums/enums";
import Button from "@/components/ui/CustomButton/Button/Button";
import { useTranslation } from "react-i18next";
import { IEditPrivacySettings } from "@/modules/settings/PrivacySettings/types/types";
import { snack } from "@/common/utils/snackManager";
import { useEditPrivacySettingsMutation } from "@/modules/settings/PrivacySettings/api/api";
import { useErrors } from "@/hooks/useErrors";
import { usePrivacySettings } from "@/modules/settings/PrivacySettings/hooks/usePrivacySettings";

interface Props {
  open: boolean;
  onClose: Function;
  currentPrivacy: PRIVACY_SETTINGS_CHOICE;
  fieldName: keyof IEditPrivacySettings;
}

const PrivacyIconModal: FC<Props> = ({
  open,
  onClose,
  currentPrivacy,
  fieldName,
}) => {
  const { t } = useTranslation();
  const { privacySettings } = usePrivacySettings();
  const [updatePrivacySettings, { isLoading: LOADING_UPDATE }] =
    useEditPrivacySettingsMutation();
  const { showResponseErrors } = useErrors();

  const privacyOptions = [
    {
      value: PRIVACY_SETTINGS_CHOICE.EVERYONE,
      icon: Globe,
      gradient: "from-blue-500 to-blue-700",
      title: t(`enum.${PRIVACY_SETTINGS_CHOICE.EVERYONE}`),
      description: t("common.privacy_everyone_description"),
    },
    {
      value: PRIVACY_SETTINGS_CHOICE.MY_FRIENDS,
      icon: Users,
      gradient: "from-amber-500 to-amber-700",
      title: t(`enum.${PRIVACY_SETTINGS_CHOICE.MY_FRIENDS}`),
      description: t("common.privacy_friends_description"),
    },
    {
      value: PRIVACY_SETTINGS_CHOICE.NOBODY,
      icon: Lock,
      gradient: "from-violet-500 to-violet-700",
      title: t(`enum.${PRIVACY_SETTINGS_CHOICE.NOBODY}`),
      description: t("common.privacy_nobody_description"),
    },
  ];

  const [privacy, setPrivacy] =
    useState<PRIVACY_SETTINGS_CHOICE>(currentPrivacy);

  const handleClose = () => {
    setPrivacy(currentPrivacy);
    onClose();
  };

  const submitSelect = async () => {
    try {
      await updatePrivacySettings({
        _id: privacySettings!._id,
        [fieldName]: privacy,
      }).unwrap();

      onClose();
      snack.success(t("user_messages.information_updated"));
    } catch (error) {
      showResponseErrors(error);
    }
  };

  return (
    <Modal open={open} onClose={handleClose}>
      <div className="w-[95%] max-w-[480px] bg-(--bg-color) rounded-2xl shadow-(--card-shadow) overflow-y-auto animate-slide-up duration-300">
        {/* Header */}
        <div className="flex flex-col p-6 border-b border-(--border-color) sm:p-7">
          <div className="flex items-center justify-between mb-2">
            <h2 className="flex items-center gap-2.5 text-xl font-bold text-(--text-color) m-0">
              <Shield size={24} className="text-(--primary-color)" />
              {t("common.privacy_title")}
            </h2>

            {/* Yeni Close Button */}
            <Button
              onClick={handleClose}
              className="flex items-center justify-center w-8 h-8 transition-all text-(--muted-color) hover:bg-black/5 dark:hover:bg-white/5 hover:text-(--text-color)"
              aria-label="Close"
            >
              <X size={18} />
            </Button>
          </div>
          <p className="m-0 text-sm font-medium text-(--muted-color)">
            {t("common.privacy_subtitle")}
          </p>
        </div>

        {/* Body */}
        <div className="p-5 sm:p-7">
          <div className="flex flex-col gap-3 mb-6">
            {privacyOptions.map((option) => {
              const Icon = option.icon;
              const isActive = privacy === option.value;

              return (
                <div
                  key={option.value}
                  onClick={() => setPrivacy(option.value)}
                  className={`group flex items-start gap-4 p-4 rounded-xl cursor-pointer transition-all border-2 
                    ${
                      isActive
                        ? "bg-(--active-bg) border-(--primary-color) shadow-(--active-shadow)"
                        : "bg-(--bg-color) border-transparent shadow-(--card-shadow) hover:bg-(--active-bg-2) hover:-translate-y-0.5 duration-500"
                    }`}
                >
                  {/* Icon Box */}
                  <div
                    className={`flex items-center justify-center shrink-0 w-11 h-11 rounded-xl bg-linear-to-br transition-transform duration-500 ${option.gradient} ${isActive ? "scale-110 shadow-lg shadow-black/20" : ""}`}
                  >
                    <Icon size={22} className="text-white" />
                  </div>

                  {/* Content */}
                  <div className="flex-1">
                    <div className="flex items-center justify-between mb-1">
                      <h3 className="m-0 text-base font-bold text-(--text-color)">
                        {option.title}
                      </h3>

                      {/* Custom Checkbox */}
                      <div
                        className={`flex items-center justify-center w-5 h-5 rounded-full border-2 transition-all 
                        ${
                          isActive
                            ? "bg-(--primary-color) border-(--primary-color)"
                            : "border-(--border-color)"
                        }`}
                      >
                        {isActive && (
                          <Check size={12} className="text-white stroke-3" />
                        )}
                      </div>
                    </div>
                    <p className="m-0 text-sm leading-relaxed text-(--muted-color)">
                      {option.description}
                    </p>
                  </div>
                </div>
              );
            })}
          </div>

          <Button
            onClick={submitSelect}
            className="w-full py-3 font-bold transition-all rounded-xl bg-(--primary-color) translate-y-0 hover:-translate-y-0.5 duration-500"
            disabled={currentPrivacy === privacy || LOADING_UPDATE}
            isLoading={LOADING_UPDATE}
          >
            {t("common.save")}
          </Button>
        </div>
      </div>
    </Modal>
  );
};

export default PrivacyIconModal;

```

## File: connectfy-client/src/modules/profile/ui/components/Modal/PersonalInfoModal/PersonalInfoModal.tsx
```typescript
import { FC } from "react";
import { useTranslation } from "react-i18next";
import { useFormik } from "formik";
import { User, MapPin } from "lucide-react";
import Modal from "@/components/Modal";
import Button from "@/components/ui/CustomButton/Button/Button";
import Input from "@/components/ui/CustomInput/Input/Input";
import DatePicker from "@/components/ui/DatePicker/DatePicker";
import NativeSelect, {
  ISelectOption,
} from "@/components/ui/Select/NativeSelect/NativeSelect";
import { GENDER } from "@/common/enums/enums";
import { checkEmptyString } from "@/common/utils/checkValues";
import { useErrors } from "@/hooks/useErrors";
import useFormDisabled from "@/hooks/useFormDisabled";
import { useUpdateProfileMutation } from "@/modules/profile/api/api";
import { validateEditProfile } from "@/modules/profile/constants/validation";
import { IAccount, IEditProfile } from "@/modules/profile/types/types";
import { getChangedData } from "@/common/utils/getDirtyValues";
import { snack } from "@/common/utils/snackManager";

interface IProps {
  open: boolean;
  onClose: () => void;
  profile: IAccount | undefined;
}

const PersonalInfoModal: FC<IProps> = ({ open, onClose, profile }) => {
  const { t } = useTranslation();
  const { showResponseErrors, showFormikErrors } = useErrors();
  const [update, { isLoading }] = useUpdateProfileMutation();

  const genderOptions: ISelectOption[] = [
    { label: t("enum.male"), value: GENDER.MALE },
    { label: t("enum.female"), value: GENDER.FEMALE },
    { label: t("enum.other"), value: GENDER.OTHER },
  ];

  const initialState: IEditProfile = {
    _id: profile?._id ?? "",
    firstName: profile?.firstName,
    lastName: profile?.lastName,
    gender: profile?.gender,
    location: profile?.location,
    birthdayDate: profile?.birthdayDate,
  };

  const formik = useFormik({
    initialValues: initialState,
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    validate: (values) => validateEditProfile(values, t),
    onSubmit: async (values) => {
      try {
        values.location = values.location?.trim() || null;
        const changedData = getChangedData<IEditProfile>(profile!, values);

        const finalData = {
          _id: profile?._id as string,
          ...changedData,
        };

        await update(finalData).unwrap();
        snack.success(t("user_messages.information_updated"));
        handleClose();
      } catch (error) {
        showResponseErrors(error);
      }
    },
  });

  const isDisabled = useFormDisabled<IEditProfile>({
    formik,
    loading: isLoading,
    validationRules: [
      (values) => {
        const requiredStringFields = [
          values.firstName,
          values.lastName,
          values.gender,
        ];

        const areStringsValid = requiredStringFields.every(
          (f) => f && checkEmptyString(f),
        );

        const location = values.location?.trim() || null;
        const isLocationValid = values.location === location;

        const isDateValid = !!values.birthdayDate;

        return areStringsValid && isLocationValid && isDateValid;
      },
      formik.dirty,
    ],
  });

  const handleClose = () => {
    formik.resetForm();
    onClose();
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    const errors = await formik.validateForm();
    if (Object.keys(errors).length > 0) {
      showFormikErrors(errors);
      return;
    }
    formik.handleSubmit(e as any);
  };

  return (
    <Modal open={open} onClose={handleClose}>
      <div className="w-full max-w-[550px] bg-(--auth-main-bg) rounded-2xl shadow-(--card-shadow) overflow-hidden animate-slide-up duration-600">
        {/* Modal Header */}
        <div className="p-6 border-b border-(--auth-glass-border)">
          <h2 className="text-xl font-bold text-(--text-primary) flex items-center gap-2">
            <User size={22} className="text-(--primary-color)" />
            {t("common.personal_information")}
          </h2>
          <p className="text-sm text-(--muted-color) mt-1">
            {t("common.edit_info_description")}
          </p>
        </div>

        {/* Modal Body */}
        <form onSubmit={handleSubmit} className="p-6">
          <div className="flex flex-col gap-6">
            {/* Name & Last Name Grid */}
            <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
              <Input
                name="firstName"
                title={t("common.first_name")}
                isFloating
                icon={<User size={18} />}
                value={formik.values.firstName || ""}
                onChange={formik.handleChange}
                isError={!!formik.errors.firstName}
              />
              <Input
                name="lastName"
                title={t("common.last_name")}
                isFloating
                icon={<User size={18} />}
                value={formik.values.lastName || ""}
                onChange={formik.handleChange}
                isError={!!formik.errors.lastName}
              />
            </div>

            {/* Birthday & Gender Grid */}
            <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
              <div className="flex flex-col">
                <DatePicker
                  value={formik.values.birthdayDate?.toString() || ""}
                  onChange={(date) =>
                    formik.setFieldValue("birthdayDate", date)
                  }
                  inputSize="large"
                  hasError={
                    !!(
                      formik.errors.birthdayDate && formik.touched.birthdayDate
                    )
                  }
                  title={t("common.birthday")}
                />
              </div>
              <NativeSelect
                title={t("common.gender")}
                inputSize="large"
                icon={<User size={18} />}
                options={genderOptions}
                value={formik.values.gender || GENDER.MALE}
                onChange={(val) => formik.setFieldValue("gender", val)}
                disabled={isLoading}
              />
            </div>

            {/* Location Input */}
            <Input
              name="location"
              title={t("common.location")}
              isFloating
              icon={<MapPin size={18} />}
              value={formik.values.location || ""}
              onChange={(e) => {
                const value = e.target.value || null;
                formik.setFieldValue("location", value);
              }}
              isError={!!formik.errors.location}
            />
          </div>

          {/* Action Buttons */}
          <div className="flex items-center gap-3 mt-8">
            <Button
              type="button"
              className="flex-1 h-12 rounded-xl text-sm font-bold bg-black/5 dark:bg-white/5 text-(--text-primary) hover:bg-black/10 dark:hover:bg-white/10 transition-all"
              onClick={handleClose}
              disabled={isLoading}
              title={t("common.cancel")}
            />
            <Button
              type="submit"
              className="flex-1 h-12 rounded-xl text-sm font-bold bg-linear-to-r from-(--primary-color) to-(--third-color) text-white shadow-lg shadow-(--primary-color)/20 transition-all duration-600"
              disabled={isDisabled}
              isLoading={isLoading}
              title={t("common.save_changes")}
            />
          </div>
        </form>
      </div>
    </Modal>
  );
};

export default PersonalInfoModal;

```

## File: connectfy-client/src/modules/profile/ui/components/PersonalInformation/PersonalInformation.tsx
```typescript
import {
  Cake,
  Phone,
  Mail,
  MapPin,
  MapMinusIcon,
  Contact,
  Users,
  UserPen,
  PhoneOff,
  MailMinus,
  CircleOff,
  CalendarMinus,
} from "lucide-react";
import { FC, Fragment } from "react";
import { useTranslation } from "react-i18next";
import { DDMMMMYYYY, formatPhoneNumber } from "@/common/utils/formatValues";
import { PRIVACY_SETTINGS_CHOICE } from "@/common/enums/enums";
import { PrivacyIcon } from "../PrivacyIcon/PrivacyIcon";
import Button from "@/components/ui/CustomButton/Button/Button";
import {
  IEditPrivacySettings,
  IPrivacySettings,
} from "@/modules/settings/PrivacySettings/types/types";
import { COUNTRIES } from "@/common/constants/constants";
import useBoolean from "@/hooks/useBoolean";
import PersonalInfoModal from "../Modal/PersonalInfoModal/PersonalInfoModal";
import { useIsMobile } from "@/hooks/useIsMobile";
import { IAccount, IMe } from "@/modules/profile/types/types";

export interface IInfoItem {
  id: string;
  icon: React.ReactNode;
  label: string;
  value: string | React.ReactNode;
  colorClass: string;
  privacyField: keyof IEditPrivacySettings;
}

interface IProps {
  user: IMe | undefined;
  profile: IAccount | undefined;
  privacySettings: IPrivacySettings | undefined;
  hasPhoneNumber: boolean;
}

const PersonalInformation: FC<IProps> = ({
  user,
  profile,
  privacySettings,
  hasPhoneNumber,
}) => {
  const { t } = useTranslation();
  const { open, onOpen, onClose } = useBoolean();
  const isMobile = useIsMobile();

  const phoneNumberMask = COUNTRIES.find(
    (country) => country.code === user?.phoneNumber?.countryCode,
  )?.format;

  // Məlumat kartlarını dinamik render etmək üçün obyektlər massivi
  const infoItems: IInfoItem[] = [
    {
      id: "email",
      icon: <Mail size={isMobile ? 15 : 20} />,
      label: t("common.email"),
      value: user?.email || (
        <span className="flex items-center gap-1 italic font-medium text-slate-400">
          <MailMinus size={16} /> {t("common.no_email_data")}
        </span>
      ),
      colorClass:
        "bg-indigo-100 text-indigo-600 dark:bg-indigo-500/15 dark:text-indigo-400",
      privacyField: "email",
    },
    {
      id: "gender",
      icon: <Users size={isMobile ? 15 : 20} />,
      label: t("common.gender"),
      value: profile?.gender ? (
        t(`enum.${profile.gender.toLowerCase()}`)
      ) : (
        <span className="flex items-center gap-1 italic font-medium text-slate-400">
          <CircleOff size={16} /> {t("common.no_gender_data")}
        </span>
      ),
      colorClass:
        "bg-purple-100 text-purple-600 dark:bg-purple-500/15 dark:text-purple-400",
      privacyField: "gender",
    },
    {
      id: "phone",
      icon: <Phone size={isMobile ? 15 : 20} />,
      label: t("common.phoneNumber"),
      value: hasPhoneNumber ? (
        formatPhoneNumber(
          user?.phoneNumber?.number as string,
          phoneNumberMask as string,
          user?.phoneNumber?.countryCode as string,
        )
      ) : (
        <span className="flex items-center gap-1 italic font-medium text-slate-400">
          <PhoneOff size={16} /> {t("common.no_phone_data")}
        </span>
      ),
      colorClass:
        "bg-emerald-100 text-emerald-600 dark:bg-emerald-500/15 dark:text-emerald-400",
      privacyField: "phoneNumber",
    },
    {
      id: "birthday",
      icon: <Cake size={isMobile ? 15 : 20} />,
      label: t("common.birthday"),
      value: profile?.birthdayDate ? (
        DDMMMMYYYY(profile?.birthdayDate)
      ) : (
        <span className="flex items-center gap-1 italic font-medium text-slate-400">
          <CalendarMinus size={16} /> {t("common.no_birthday_data")}
        </span>
      ),
      colorClass:
        "bg-orange-100 text-orange-600 dark:bg-orange-500/15 dark:text-orange-400",
      privacyField: "birthdayDate",
    },
    {
      id: "location",
      icon: <MapPin size={isMobile ? 15 : 20} />,
      label: t("common.location"),
      value: profile?.location || (
        <span className="flex items-center gap-1 italic font-medium text-slate-400">
          <MapMinusIcon size={16} /> {t("common.no_location_info")}
        </span>
      ),
      colorClass:
        "bg-emerald-100 text-emerald-600 dark:bg-emerald-500/15 dark:text-emerald-400",
      privacyField: "location",
    },
  ];

  return (
    <Fragment>
      <section
        className="p-8 mb-8 px-4 md:px-8 transition-all duration-300 bg-(--auth-main-bg) rounded-[20px] border border-(--auth-glass-border) shadow-(--card-shadow)"
        aria-labelledby="personal-info-heading"
      >
        {/* Header */}
        <div className="flex flex-row items-center justify-between gap-3 mb-6 md:mb-8">
          {/* Sol tərəf: İkon və Başlıq */}
          <div className="flex items-center gap-2 md:gap-3">
            <Contact className="w-6 h-6 md:w-7 md:h-7 text-(--primary-color)" />
            <h2
              id="personal-info-heading"
              className="m-0 text-lg font-bold md:text-2xl text-(--text-color)"
            >
              {t("common.personal_information")}
            </h2>
          </div>

          {/* Sağ tərəf: Düymə */}
          <div className="flex items-center gap-2 md:gap-3">
            {/* Böyük ekranlar üçün düymə (1024px-dən yuxarı görünür) */}
            <Button
              className="hidden lg:flex bg-(--btn-edit-bg)! text-(--btn-edit-text)! font-semibold px-4 py-2 rounded-lg items-center gap-2 transition-all hover:opacity-80 border-none text-sm whitespace-nowrap"
              title={t("common.edit")}
              onClick={onOpen}
            />

            {/* Mobil ekranlar üçün ikonlu düymə (1024px-dən aşağı görünür) */}
            <Button
              className="flex lg:hidden bg-(--btn-edit-bg)! text-(--btn-edit-text)! font-semibold p-3 rounded-lg items-center justify-center transition-all hover:opacity-80 border-none"
              icon={<UserPen size={16} />}
              onClick={onOpen}
            />
          </div>
        </div>

        {/* Grid */}

        <div className="grid grid-cols-1 gap-4 sm:grid-cols-2">
          {infoItems.map((item) => (
            <div
              key={item.id}
              className="flex items-center gap-4 p-4 lg:px-5 bg-(--info-card-bg) border border-(--info-card-border) rounded-xl transition-all duration-200 group"
            >
              {/* Icon Box */}
              <div
                className={`flex items-center justify-center rounded-xl shrink-0 transition-transform w-10 h-10 md:w-12 md:h-12 ${item.colorClass}`}
              >
                {item.icon}
              </div>

              {/* Content */}
              <div className="flex flex-col flex-1 gap-1 overflow-hidden">
                <span className="text-[11px] font-bold tracking-widest text-(--muted-color) uppercase">
                  {item.label}
                </span>
                <div className="text-[15px] font-semibold text-(--text-color) truncate flex items-center gap-1.5">
                  {item.value}
                </div>
              </div>

              {/* Privacy Icon */}
              <div className="flex items-center justify-center text-(--text-disabled) opacity-60">
                <PrivacyIcon
                  privacy={
                    (privacySettings?.[
                      item.privacyField
                    ] as PRIVACY_SETTINGS_CHOICE) ||
                    PRIVACY_SETTINGS_CHOICE.EVERYONE
                  }
                  fieldName={item.privacyField as keyof IEditPrivacySettings}
                />
              </div>
            </div>
          ))}
        </div>
      </section>

      {open && (
        <PersonalInfoModal open={open} onClose={onClose} profile={profile} />
      )}
    </Fragment>
  );
};

export default PersonalInformation;

```

## File: connectfy-client/src/modules/profile/ui/components/PrivacyIcon/PrivacyIcon.tsx
```typescript
import { memo } from "react";
import { Globe, Users, Lock } from "lucide-react";
import { PRIVACY_SETTINGS_CHOICE } from "@/common/enums/enums";
import useBoolean from "@/hooks/useBoolean";
import PrivacyIconModal from "../Modal/PrivacyIconModal/PrivacyIconModal";
import { useTranslation } from "react-i18next";
import { IEditPrivacySettings } from "@/modules/settings/PrivacySettings/types/types";
import Button from "@/components/ui/CustomButton/Button/Button";
import TextTooltip from "@/components/Tooltip/TextTooltip";

interface Props {
  privacy: PRIVACY_SETTINGS_CHOICE;
  fieldName: keyof IEditPrivacySettings;
}

export const PrivacyIcon = memo(({ privacy, fieldName }: Props) => {
  const { t } = useTranslation();
  const { open, onOpen, onClose } = useBoolean();

  // İkonlar üçün ortaq class
  const iconClassName = "text-(--text-color) transition-all duration-300";

  const icons = {
    [PRIVACY_SETTINGS_CHOICE.EVERYONE]: (
      <Globe size={16} className={iconClassName} />
    ),
    [PRIVACY_SETTINGS_CHOICE.MY_FRIENDS]: (
      <Users size={16} className={iconClassName} />
    ),
    [PRIVACY_SETTINGS_CHOICE.NOBODY]: (
      <Lock size={16} className={iconClassName} />
    ),
  };

  return (
    <>
      <TextTooltip position="top" text={t(`enum.${privacy}`)}>
        <Button
          type="button"
          onClick={onOpen}
          className="group flex items-center justify-center p-1.5 rounded-lg transition-colors hover:bg-black/5 dark:hover:bg-white/5 cursor-pointer border-none bg-transparent"
        >
          {icons[privacy]}
        </Button>
      </TextTooltip>

      <PrivacyIconModal
        open={open}
        onClose={onClose}
        currentPrivacy={privacy}
        fieldName={fieldName}
      />
    </>
  );
});

```

## File: connectfy-client/src/modules/profile/ui/components/MainCard/MainCard.tsx
```typescript
import { Pencil, UserRoundX, Users } from "lucide-react";
import { useTranslation } from "react-i18next";
import Button from "@/components/ui/CustomButton/Button/Button";
import { FC, Fragment } from "react";
import { IAccount, IMe } from "@/modules/profile/types/types";
import NoProfilePhotoIcon from "@/assets/icons/NoProfilePhotoIcon";
import { useAvatarModalStore } from "@/store/zustand/useAvatarModalStore";
import { showDate } from "@/common/utils/formatValues";
import { DATE_FORMAT } from "@/common/enums/enums";
import { useGeneralSettings } from "@/modules/settings/GeneralSettings/hooks/useGeneralSettings";

interface IProps {
  user: IMe | undefined;
  profile: IAccount | undefined;
}

const MainCard: FC<IProps> = ({ user, profile }) => {
  const { t } = useTranslation();

  const { generalSettings } = useGeneralSettings();

  const onOpenShowModal = useAvatarModalStore((state) => state.onOpenShowModal);
  const onOpenChangeModal = useAvatarModalStore(
    (state) => state.onOpenChangeModal,
  );

  return (
    <Fragment>
      <section
        className="flex flex-col items-center gap-6 p-8 mb-8 transition-all bg-(--auth-main-bg) rounded-[20px] border border-(--auth-glass-border) shadow-(--card-shadow) md:p-10"
        aria-labelledby="profile-main-heading"
      >
        <div className="relative shrink-0">
          <div className="size-[140px] xs:size-[100px] sm:size-[120px] md:size-[140px] rounded-full overflow-hidden shadow-[0_8px_24px_var(--shadow-color)] border-4 border-(--primary-color)">
            {profile?.avatar?.url ? (
              <img
                src={profile.avatar.url}
                alt={`Profile picture of ${profile?.fullName || `${profile?.firstName} ${profile?.lastName}`}`}
                loading="eager"
                fetchPriority="high"
                decoding="async"
                className="object-cover w-full h-full bg-(--skeleton-card-bg) cursor-pointer"
                onClick={() =>
                  onOpenShowModal(
                    profile?.avatar?.url ?? "",
                    user?.username,
                    user?._id,
                    profile?._id,
                  )
                }
              />
            ) : (
              <div className="flex items-center justify-center w-full h-full bg-(--active-bg-2)">
                <NoProfilePhotoIcon />
              </div>
            )}
          </div>

          <Button
            type="button"
            onClick={() => onOpenChangeModal(profile?.avatar, profile?._id)}
            className="absolute bottom-1 right-1 md:bottom-0 md:right-0 flex items-center justify-center w-10 h-10 md:w-11 md:h-11 bg-(--primary-color) border-4 border-(--auth-main-bg) rounded-full shadow-md text-white hover:bg-(--hover-bg) transition-colors"
            icon={<Pencil size={18} />}
          />
        </div>

        <div className="text-center">
          <h1
            id="profile-main-heading"
            className="m-0 mb-2 text-2xl font-bold leading-tight md:text-[28px] text-(--text-color)"
          >
            {profile?.fullName || `${profile?.firstName} ${profile?.lastName}`}
          </h1>
          <p className="m-0 mb-2 text-sm font-medium md:text-base text-(--muted-color)">
            @{user?.username}
          </p>
          <p className="m-0 text-sm italic text-(--muted-color)">
            {t("common.lastSeen")}:{" "}
            {profile?.lastSeen
              ? showDate(
                  profile.lastSeen,
                  generalSettings?.timeZone.dateFormat || DATE_FORMAT.DDMMYYYY,
                  "/",
                )
              : "-"}
          </p>
        </div>

        <div className="flex flex-col items-center w-full max-w-[400px] gap-4 p-4 rounded-2xl bg-(--active-bg-2) sm:flex-row sm:gap-6 sm:p-6 md:px-8">
          <div className="flex flex-row items-center justify-center flex-1 gap-3 sm:flex-col sm:gap-1.5 text-(--text-color)">
            <div className="flex items-center gap-1.5">
              <Users
                size={18}
                className="text-(--primary-color)"
                aria-hidden="true"
              />
              <span className="text-xl font-bold md:text-2xl text-(--primary-color)">
                15
              </span>
            </div>
            <span className="text-[13px] font-medium uppercase tracking-wider text-(--muted-color)">
              {t("common.friends")}
            </span>
          </div>

          <div
            className="w-full h-px opacity-30 bg-(--border-color) sm:w-px sm:h-10"
            aria-hidden="true"
          />

          <div className="flex flex-row items-center justify-center flex-1 gap-3 sm:flex-col sm:gap-1.5 text-(--text-color)">
            <div className="flex items-center gap-1.5">
              <UserRoundX
                size={18}
                className="text-(--error-color)"
                aria-hidden="true"
              />
              <span className="text-xl font-bold md:text-2xl text-(--primary-color)">
                5
              </span>
            </div>
            <span className="text-[13px] font-medium uppercase tracking-wider text-(--muted-color)">
              {t("common.blocked")}
            </span>
          </div>
        </div>
      </section>
    </Fragment>
  );
};

export default MainCard;

```

## File: connectfy-client/src/modules/profile/ui/components/ProfileHeader/ProfileHeader.tsx
```typescript
import { ArrowLeft, User, UserLock, Users, Settings } from "lucide-react";
import { ROUTER } from "@/common/constants/routet";
import { useTranslation } from "react-i18next";
import Button from "@/components/ui/CustomButton/Button/Button";
import { useAppNavigation } from "@/hooks/useAppNavigation";
import { useIsMobile } from "@/hooks/useIsMobile";
import { FC } from "react";

interface IProps {
  onToggleFriends?: () => void;
}

const ProfileHeader: FC<IProps> = ({ onToggleFriends }) => {
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();
  const isMobile = useIsMobile();

  const handleFriendsClick = () => {
    if (isMobile) {
      navigate(ROUTER.USERS.FRIENDS);
    } else {
      onToggleFriends?.();
    }
  };

  return (
    <header
      className="
        sticky top-0 left-0 right-0 z-100
        flex items-center justify-between
        gap-5 px-4 min-h-[72px]
        bg-(--auth-main-bg)
        backdrop-blur-[10px]
        max-sm:px-3 max-sm:gap-3 max-sm:min-h-[64px]
      "
    >
      {/* Left */}
      <div className="flex items-center gap-4 flex-1 min-w-0 max-sm:gap-3 max-sm:flex-none">
        <Button
          aria-label={t("common.back")}
          onClick={() => navigate(-1)}
          icon={<ArrowLeft size={18} />}
          className="
            flex items-center justify-center shrink-0 p-0
            w-[43px] h-[43px] rounded-[10px]
            bg-transparent text-(--text-color) cursor-pointer
            transition-all duration-300 ease-in-out
            hover:bg-(--active-bg) hover:border-(--primary-color) hover:-translate-x-0.5
            active:translate-x-0
            focus-visible:outline-2 focus-visible:outline-(--primary-color) focus-visible:-outline-offset-0.5
            max-[768px]:w-9 max-[768px]:h-9 max-[768px]:min-w-9 max-[768px]:min-h-9
            max-[480px]:w-8 max-[480px]:h-8 max-[480px]:min-w-8 max-[480px]:min-h-8 max-[480px]:rounded-lg
          "
        />
        <h1
          className="
            flex items-center gap-2.5 m-0 p-0
            text-xl font-bold text-(--text-color)
            whitespace-nowrap overflow-hidden text-ellipsis
            [&>svg]:text-(--primary-color) [&>svg]:stroke-[2.5] [&>svg]:shrink-0 [&>svg]:w-[18px]
            max-[768px]:text-lg max-[768px]:gap-2
            max-[640px]:text-base max-[640px]:shrink
            max-[480px]:text-base max-[480px]:gap-1.5 max-[480px]:[&>svg]:w-4 max-[480px]:[&>svg]:h-4
          "
        >
          <User size={20} />
          <span>{t("common.profile")}</span>
        </h1>
      </div>

      {/* Right */}
      <div className="flex items-center gap-3 max-sm:flex-1 max-sm:justify-end max-sm:min-w-0">
        {/* Blocklist — error rengi (first-of-type) */}
        <Button
          onClick={() => navigate(ROUTER.USERS.BLOCKLIST)}
          icon={<UserLock size={20} />}
          title={t("common.blocklist")}
          className="
            flex items-center gap-2 px-4 h-10 min-h-10 rounded-xl
            border-none cursor-pointer font-medium text-sm whitespace-nowrap
            shadow-(--card-shadow)
            bg-[rgba(var(--error-color-rgb),0.1)] text-(--error-color)
            transition-all duration-300 ease-in-out
            hover:bg-(--error-color) hover:text-white hover:-translate-y-0.5 hover:shadow-(--active-shadow)
            active:translate-y-0
            focus-visible:outline-2 focus-visible:outline-(--primary-color) focus-visible:-outline-offset-0.5
            max-[768px]:px-3 max-[768px]:h-9 max-[768px]:min-h-9
            max-[480px]:px-2.5 max-[480px]:h-8 max-[480px]:min-h-8 max-[480px]:gap-1.5
          "
        />

        {/* Friends */}
        <Button
          onClick={handleFriendsClick}
          icon={<Users size={20} />}
          title={t("common.my_friends")}
          className="
            flex items-center gap-2 px-4 h-10 min-h-10 rounded-xl
            border-none cursor-pointer font-medium text-sm whitespace-nowrap
            shadow-(--card-shadow)
            bg-[rgba(var(--primary-color-rgb),0.1)] text-(--primary-color)
            transition-all duration-300 ease-in-out
            hover:bg-(--primary-color) hover:text-white hover:-translate-y-0.5 hover:shadow-(--active-shadow)
            active:translate-y-0
            focus-visible:outline-2 focus-visible:outline-(--primary-color) focus-visible:-outline-offset-0.5
            max-[768px]:px-3 max-[768px]:h-9 max-[768px]:min-h-9
            max-[480px]:px-2.5 max-[480px]:h-8 max-[480px]:min-h-8 max-[480px]:gap-1.5
          "
        />

        {/* Settings */}
        <Button
          onClick={() => navigate(ROUTER.SETTINGS.MAIN)}
          icon={<Settings size={20} />}
          title={t("common.settings")}
          className="
            flex items-center gap-2 px-4 h-10 min-h-10 rounded-xl
            border-none cursor-pointer font-medium text-sm whitespace-nowrap
            shadow-(--card-shadow)
            bg-[rgba(var(--primary-color-rgb),0.1)] text-(--primary-color)
            transition-all duration-300 ease-in-out
            hover:bg-(--primary-color) hover:text-white hover:-translate-y-0.5 hover:shadow-(--active-shadow)
            active:translate-y-0
            focus-visible:outline-2 focus-visible:outline-(--primary-color) focus-visible:-outline-offset-0.5
            max-[768px]:px-3 max-[768px]:h-9 max-[768px]:min-h-9
            max-[480px]:px-2.5 max-[480px]:h-8 max-[480px]:min-h-8 max-[480px]:gap-1.5
          "
        />
      </div>
    </header>
  );
};

export default ProfileHeader;

```

## File: connectfy-client/src/modules/profile/ui/components/SocialLinks/SocialLinks.tsx
```typescript
import { FC, Fragment, useCallback, useMemo, useState } from "react";
import { useTranslation } from "react-i18next";
import { Link2, GripVertical } from "lucide-react";
import { motion, AnimatePresence, Reorder } from "framer-motion";

import { usePrivacySettings } from "@/modules/settings/PrivacySettings/hooks/usePrivacySettings";
import { useAuthStore } from "@/store/zustand/useAuthStore";
import { useErrors } from "@/hooks/useErrors";
import useBoolean from "@/hooks/useBoolean";
import { ISocialLink } from "@/modules/profile/types/types";
import { PRIVACY_SETTINGS_CHOICE } from "@/common/enums/enums";
import { snack } from "@/common/utils/snackManager";

import {
  useGetSocialLinksQuery,
  useRemoveSocialLinkMutation,
  useRemoveSocialLinksMutation,
  useUpdateSocialLinkRankMutation,
} from "@/modules/profile/api/api";

import SocialLinkSkeleton from "@/components/Skeleton/profile/SocialLinkSkeleton";
import ActionConfirmModal from "@/components/Modal/ActionConfirmModal/ActionConfirmModal";
import Checkbox from "@/components/ui/CustomCheckbox/Checkbox/Checkbox";

import SocialLinkItem from "./components/SocialLinkItem";
import AddSocialLinkModal from "./components/Modal/AddSocialLink";
import EditSocialLinkModal from "./components/Modal/EditSocialLink";
import SocialLinkHeader from "./components/SocialLinkHeader";
import { useContextMenu } from "@/hooks/useContextMenu";
import SocialContextMenu from "./components/SocialLinkContextMenu";

interface IProps {
  userId: string | undefined;
}

const SocialLinks: FC<IProps> = ({ userId }) => {
  const { t } = useTranslation();
  const { privacySettings } = usePrivacySettings();
  const { access_token } = useAuthStore();
  const { showResponseErrors } = useErrors();

  // States
  const [selectedLink, setSelectedLink] = useState<ISocialLink | null>(null);
  const [activeFilter, setActiveFilter] = useState<
    "rank" | "newest" | "oldest"
  >("rank");

  // Modes
  const [isSelectionMode, setIsSelectionMode] = useState<boolean>(false);
  const [selectedIds, setSelectedIds] = useState<string[]>([]);
  const [isReorderMode, setIsReorderMode] = useState<boolean>(false);
  const [reorderLinks, setReorderLinks] = useState<ISocialLink[]>([]);

  // Modals
  const addModal = useBoolean();
  const editModal = useBoolean();
  const removeOneModal = useBoolean();
  const removeMultipleModal = useBoolean();

  const { handleContextMenu } = useContextMenu();

  // API Hooks
  const { data: socialLinks, isLoading } = useGetSocialLinksQuery(
    { userId: userId || "" },
    { skip: !access_token || !userId },
  );

  const [removeOne, { isLoading: isRemoveOneLoading }] =
    useRemoveSocialLinkMutation();
  const [removeMany, { isLoading: isRemoveManyLoading }] =
    useRemoveSocialLinksMutation();
  const [updateRank, { isLoading: isUpdateRankLoading }] =
    useUpdateSocialLinkRankMutation();

  const sortedData = useMemo(() => {
    if (!socialLinks?.data) return [];

    const links = [...socialLinks.data];

    switch (activeFilter) {
      case "newest":
        return links.sort(
          (a, b) =>
            new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime(),
        );
      case "oldest":
        return links.sort(
          (a, b) =>
            new Date(a.createdAt).getTime() - new Date(b.createdAt).getTime(),
        );
      case "rank":
      default:
        return links.sort((a, b) => (a.rank || 0) - (b.rank || 0));
    }
  }, [socialLinks?.data, activeFilter]);

  // Actions
  const toggleSelectionMode = useCallback(() => {
    setIsSelectionMode((prev) => !prev);
    setIsReorderMode(false);
    setSelectedIds([]);
  }, []);

  const toggleReorderMode = useCallback(() => {
    setIsReorderMode((prev) => {
      if (!prev) setReorderLinks(sortedData);
      return !prev;
    });
    setIsSelectionMode(false);
  }, [sortedData]);

  const orderedLinks = isReorderMode ? reorderLinks : sortedData;

  const toggleSelect = useCallback((id: string) => {
    setSelectedIds((prev) =>
      prev.includes(id) ? prev.filter((item) => item !== id) : [...prev, id],
    );
  }, []);

  const handleSocialAction = useCallback(
    (action: string, link: ISocialLink) => {
      if (isSelectionMode || isReorderMode) return;
      switch (action) {
        case "openLink":
          window.open(link.url, "_blank");
          break;
        case "copy":
          navigator.clipboard.writeText(link.url);
          snack.success(t("common.link_copied"));
          break;
        case "edit":
          setSelectedLink(link);
          editModal.onOpen();
          break;
        case "delete":
          setSelectedLink(link);
          removeOneModal.onOpen();
          break;
        case "handleSelect":
          toggleSelectionMode();
          toggleSelect(link._id);
          break;
      }
    },
    [
      isSelectionMode,
      isReorderMode,
      editModal,
      removeOneModal,
      toggleSelect,
      toggleSelectionMode,
      t,
    ],
  );

  const handleSaveOrder = async () => {
    try {
      const payload = orderedLinks.map((link, index) => ({
        _id: link._id,
        rank: index + 1,
      }));

      await updateRank({
        links: payload,
        userId: userId || "",
      }).unwrap();

      setIsReorderMode(false);
      snack.success(t("user_messages.information_updated"));
    } catch (error) {
      showResponseErrors(error);
    }
  };

  const handleRemoveOne = async () => {
    if (!selectedLink) return;
    try {
      await removeOne({ _id: selectedLink._id }).unwrap();
      removeOneModal.onClose();
      setSelectedLink(null);
      snack.success(t("user_messages.information_removed"));
    } catch (error) {
      showResponseErrors(error);
    }
  };

  const handleRemoveMultiple = async () => {
    if (selectedIds.length === 0) return;
    try {
      const res = await removeMany({
        _ids: selectedIds,
        userId: userId || "",
      }).unwrap();
      removeMultipleModal.onClose();
      setIsSelectionMode(false);
      setSelectedIds([]);
      snack.success(
        t("user_messages.information_removed_count", {
          count: res.deletedCount,
        }),
      );
    } catch (error) {
      showResponseErrors(error);
    }
  };

  const hasLinks = orderedLinks && orderedLinks.length > 0;
  const currentPrivacy =
    privacySettings?.socialLinks || PRIVACY_SETTINGS_CHOICE.EVERYONE;

  return (
    <Fragment>
      {isLoading ? (
        <SocialLinkSkeleton />
      ) : (
        <section
          className="p-8 mb-8 transition-all duration-300 bg-(--auth-main-bg) rounded-[20px] border border-(--auth-glass-border) shadow-(--card-shadow)"
          aria-labelledby="social-links-heading"
        >
          {/* Ekstrakt edilmiş Header Komponenti */}
          <SocialLinkHeader
            isSelectionMode={isSelectionMode}
            isReorderMode={isReorderMode}
            hasLinks={hasLinks}
            selectedCount={selectedIds.length}
            totalCount={socialLinks?.totalCount}
            activeFilter={activeFilter}
            privacy={currentPrivacy}
            isUpdateRankLoading={isUpdateRankLoading}
            onToggleSelectionMode={toggleSelectionMode}
            onToggleReorderMode={toggleReorderMode}
            onRemoveMultiple={removeMultipleModal.onOpen}
            onSaveOrder={handleSaveOrder}
            onAddClick={addModal.onOpen}
            onFilterChange={setActiveFilter}
          />

          {/* Siyahı hissəsi */}
          {hasLinks ? (
            <Reorder.Group
              axis="y"
              values={orderedLinks}
              onReorder={setReorderLinks}
              className="flex flex-col gap-4"
            >
              {orderedLinks.map((link, index) => {
                const isSelected = selectedIds.includes(link._id);

                return (
                  <Reorder.Item
                    key={link._id}
                    value={link}
                    dragListener={isReorderMode}
                    className={`flex items-center w-full ${isReorderMode ? "cursor-grab active:cursor-grabbing" : ""}`}
                    onContextMenu={(e) => {
                      if (isSelectionMode || isReorderMode) return;

                      handleContextMenu(
                        e,
                        <SocialContextMenu
                          onCopy={() => handleSocialAction("copy", link)}
                          onEdit={() => handleSocialAction("edit", link)}
                          onOpenLink={() =>
                            handleSocialAction("openLink", link)
                          }
                          onRemove={() => {
                            setSelectedLink(link);
                            removeOneModal.onOpen();
                          }}
                          onSelect={() => {
                            setIsSelectionMode(true);
                            toggleSelect(link._id);
                          }}
                          onReorder={toggleReorderMode}
                        />,
                      );
                    }}
                  >
                    <AnimatePresence mode="popLayout">
                      {isReorderMode && (
                        <motion.div
                          initial={{ opacity: 0, width: 0, marginRight: 0 }}
                          animate={{
                            opacity: 1,
                            width: "auto",
                            marginRight: 16,
                          }}
                          exit={{ opacity: 0, width: 0, marginRight: 0 }}
                          transition={{ duration: 0.2 }}
                          className="shrink-0 flex items-center justify-center text-(--muted-color)"
                        >
                          <GripVertical size={20} />
                        </motion.div>
                      )}

                      {isSelectionMode && (
                        <motion.div
                          initial={{ opacity: 0, width: 0, marginRight: 0 }}
                          animate={{
                            opacity: 1,
                            width: "auto",
                            marginRight: 16,
                          }}
                          exit={{ opacity: 0, width: 0, marginRight: 0 }}
                          transition={{ duration: 0.2 }}
                          className="shrink-0 cursor-pointer text-(--primary-color)"
                          onClick={() => toggleSelect(link._id)}
                        >
                          <Checkbox
                            id={link._id}
                            checked={isSelected}
                            onChange={() => toggleSelect(link._id)}
                            className={`w-6! h-6! text-(--primary-color)! transition-transform! scale-110! ${isSelected ? "bg-(--primary-color)!" : "bg-(--auth-glass-bg)!"}`}
                          />
                        </motion.div>
                      )}
                    </AnimatePresence>

                    <div className="flex-1 min-w-0 transition-all duration-300">
                      <div
                        className={`${isReorderMode ? "pointer-events-none opacity-80" : ""}`}
                      >
                        <SocialLinkItem
                          link={link}
                          onAction={handleSocialAction}
                          index={index}
                          closeExpanded={isSelectionMode || isReorderMode}
                          isSelectionMode={isSelectionMode}
                          // YENİ ƏLAVƏLƏR:
                          isSelected={isSelected}
                          onToggleSelect={() => toggleSelect(link._id)}
                        />
                      </div>
                    </div>
                  </Reorder.Item>
                );
              })}
            </Reorder.Group>
          ) : (
            <div className="flex flex-col items-center justify-center p-10 border border-dashed rounded-xl border-white/10 bg-black/5">
              <Link2
                className="mb-3 opacity-20 text-(--text-color)"
                size={48}
              />
              <span className="text-(--muted-color) italic font-medium">
                {t("common.no_social_links")}
              </span>
            </div>
          )}
        </section>
      )}

      {/* Modallar */}
      {addModal.open && (
        <AddSocialLinkModal
          open={addModal.open}
          onClose={addModal.onClose}
          userId={userId || ""}
        />
      )}

      {editModal.open && selectedLink && (
        <EditSocialLinkModal
          open={editModal.open}
          onClose={() => {
            editModal.onClose();
            setSelectedLink(null);
          }}
          socialLink={selectedLink}
        />
      )}

      {removeOneModal.open && (
        <ActionConfirmModal
          open={removeOneModal.open}
          onClose={removeOneModal.onClose}
          onCancel={removeOneModal.onClose}
          onConfirm={handleRemoveOne}
          header={{ title: t("common.remove_social_link_title") }}
          cancelBtn={{ title: t("common.cancel") }}
          confirmBtn={{
            title: t("common.remove"),
            color: "var(--error-color)",
          }}
          isLoading={isRemoveOneLoading}
        >
          <p className="text-center text-[15px] text-(--muted-color)">
            {t("common.remove_social_link_desc")}
          </p>
        </ActionConfirmModal>
      )}

      {removeMultipleModal.open && (
        <ActionConfirmModal
          open={removeMultipleModal.open}
          onClose={removeMultipleModal.onClose}
          onCancel={removeMultipleModal.onClose}
          onConfirm={handleRemoveMultiple}
          header={{ title: t("common.remove_social_link_title") }}
          cancelBtn={{ title: t("common.cancel") }}
          confirmBtn={{
            title: t("common.remove"),
            color: "var(--error-color)",
          }}
          isLoading={isRemoveManyLoading}
        >
          <p className="text-center text-[15px] text-(--muted-color)">
            {t("common.remove_social_links_desc", {
              count: selectedIds.length,
            })}
          </p>
        </ActionConfirmModal>
      )}
    </Fragment>
  );
};

export default SocialLinks;

```

## File: connectfy-client/src/modules/profile/ui/components/SocialLinks/components/SocialLinkItem.tsx
```typescript
import { Link } from "react-router-dom";
import {
  ChevronDown,
  ChevronUp,
  ExternalLink,
  Copy,
  Pencil,
  Trash2,
  ListChecks,
} from "lucide-react";
import { memo, useEffect, useState } from "react";
import { ISocialLink } from "@/modules/profile/types/types";
import Button from "@/components/ui/CustomButton/Button/Button";
import { useTranslation } from "react-i18next";
import { PLATFORM_ICONS } from "./SocialLinkPlatformIcons";
import { SOCIAL_LINK_PLATFORM } from "@/common/enums/enums";

// Social Link Item Component
const SocialLinkItem = memo(
  ({
    link,
    onAction,
    index,
    closeExpanded,
    isSelectionMode,
    isSelected,
    onToggleSelect,
  }: {
    link: ISocialLink;
    onAction: (action: string, link: ISocialLink) => void;
    index: number;
    closeExpanded: boolean;
    isSelectionMode: boolean;
    isSelected: boolean;
    onToggleSelect: () => void;
  }) => {
    const { t } = useTranslation();
    const [isExpanded, setIsExpanded] = useState(false);

    const handleCopy = (e: React.MouseEvent) => {
      e.stopPropagation();
      navigator.clipboard.writeText(link.url);
      onAction("copy", link);
    };

    useEffect(() => {
      if (closeExpanded && isExpanded) {
        setIsExpanded(false);
      }
    }, [closeExpanded]);

    return (
      <div
        className={`relative overflow-hidden transition-all duration-300 border rounded-xl group
          ${
            isSelected
              ? "border-(--primary-color) bg-(--active-bg-2) ring-1 ring-(--primary-color)"
              : "bg-(--info-card-bg) border-(--info-card-border) hover:border-(--primary-color)"
          }`}
        data-testid={`social-link-item-${index}`}
      >
        {/* Accent Bar (Hover zamanı solda görünən rəngli xətt) */}
        <div
          className={`absolute top-0 left-0 w-1 h-full transition-opacity bg-(--primary-color) 
          ${isSelected ? "opacity-100" : "opacity-0 group-hover:opacity-100"}`}
        />

        {/* Header Button */}
        <Button
          onClick={() => {
            // DƏYİŞİKLİK: Selection mode aktivdirsə, seçimi dəyiş, deyilse expand et.
            if (isSelectionMode) {
              onToggleSelect();
            } else {
              setIsExpanded(!isExpanded);
            }
          }}
          className="flex items-center justify-between w-full px-5 py-4 text-left transition-colors hover:bg-(--active-bg-2)"
          aria-expanded={isExpanded}
        >
          <div className="flex items-center gap-3">
            <div
              className={`transition-colors shrink-0 ${isSelected ? "text-(--primary-color)" : "text-(--muted-color) group-hover:text-(--primary-color)"}`}
            >
              {PLATFORM_ICONS[link.platform as SOCIAL_LINK_PLATFORM]}
            </div>
            <div className="flex flex-col gap-1">
              <span className="text-[15px] font-bold text-(--text-color) tracking-wide">
                {link.platform}
              </span>
              <span className="text-sm font-medium text-(--muted-color)">
                {link.name}
              </span>
            </div>
          </div>

          {/* DƏYİŞİKLİK: Selection mode-da chevron-u gizlədə və ya checkbox göstərə bilərsən. 
              Amma sadəlik üçün chevron-u seçim zamanı gizlətmək daha yaxşıdır. */}
          {!isSelectionMode && (
            <div className="text-(--muted-color) group-hover:text-(--text-color) transition-colors">
              {isExpanded ? <ChevronUp size={20} /> : <ChevronDown size={20} />}
            </div>
          )}
        </Button>

        {/* Details (Expanded Content) */}
        <div
          className={`overflow-hidden transition-all duration-300 ease-in-out ${
            isExpanded ? "max-h-80 opacity-100" : "max-h-0 opacity-0"
          }`}
        >
          <div className="p-5 pt-0 border-t border-(--info-card-border)/50">
            {/* URL Display */}
            <Link
              to={link.url}
              target="_blank"
              rel="noopener noreferrer"
              className="block p-3 mt-4 text-sm font-medium transition-all border border-dashed rounded-lg border-white/10 text-(--primary-color) bg-white/70 dark:bg-black/30 truncate"
            >
              {link.url}
            </Link>

            {/* Action Buttons */}
            <div className="flex flex-wrap gap-2 mt-4">
              <ActionButton
                onClick={() => onAction("openLink", link)}
                icon={<ExternalLink size={16} />}
                label={t("common.open")}
              />
              <ActionButton
                onClick={handleCopy}
                icon={<Copy size={16} />}
                label={t("common.copy")}
              />
              <ActionButton
                onClick={() => onAction("edit", link)}
                icon={<Pencil size={16} />}
                label={t("common.edit")}
              />
              <ActionButton
                onClick={() => {
                  onAction("handleSelect", link);
                  setIsExpanded(false);
                }}
                icon={<ListChecks size={16} />}
                label={t("common.select")}
              />
              <ActionButton
                onClick={() => onAction("delete", link)}
                icon={<Trash2 size={16} />}
                label={t("common.remove")}
                isDelete
              />
            </div>
          </div>
        </div>
      </div>
    );
  },
);

// Köməkçi Düymə Komponenti (Daxili düymələr üçün)
const ActionButton = ({ onClick, icon, label, isDelete }: any) => (
  <Button
    onClick={onClick}
    className={`flex-1 min-w-[100px] flex items-center justify-center gap-2 px-3 py-2.5 rounded-lg text-sm font-semibold transition-all border
      ${
        isDelete
          ? "border-red-500/30 text-red-500 hover:bg-red-500 hover:text-white"
          : "border-white/5 bg-white/5 text-(--text-color) hover:bg-(--primary-color) hover:text-white hover:scale-[1.02]"
      }`}
  >
    {icon}
    <span className="hidden sm:inline">{label}</span>
  </Button>
);

export default SocialLinkItem;

```

## File: connectfy-client/src/modules/profile/ui/components/SocialLinks/components/SocialLinkContextMenu.tsx
```typescript
import { FC } from "react";
import { motion } from "framer-motion";
import {
  Pencil,
  Trash2,
  ListChecks,
  ArrowUpDown,
  Copy,
  ExternalLink,
} from "lucide-react";
import { useTranslation } from "react-i18next";
import ContextMenuItem from "@/components/ContextMenu/ContextMenuItem";

interface ContextMenuProps {
  onEdit: () => void;
  onRemove: () => void;
  onSelect: () => void;
  onReorder: () => void;
  onCopy: () => void;
  onOpenLink: () => void;
}

const SocialContextMenu: FC<ContextMenuProps> = ({
  onEdit,
  onRemove,
  onSelect,
  onReorder,
  onCopy,
  onOpenLink,
}) => {
  const { t } = useTranslation();

  return (
    <motion.div
      initial={{ opacity: 0, scale: 0.9, y: -10 }}
      animate={{ opacity: 1, scale: 1, y: 0 }}
      exit={{ opacity: 0, scale: 0.9, y: -10 }}
      transition={{ duration: 0.15, ease: "easeOut" }}
      className="flex flex-col min-w-[220px] bg-(--bg-color) rounded-2xl shadow-2xl overflow-hidden"
    >
      <ContextMenuItem
        icon={ExternalLink}
        label={t("common.open")}
        onClick={onOpenLink}
      />
      <ContextMenuItem icon={Copy} label={t("common.copy")} onClick={onCopy} />
      <ContextMenuItem
        icon={Pencil}
        label={t("common.edit")}
        onClick={onEdit}
      />

      <div className="h-px bg-white/5 mx-2" />

      <ContextMenuItem
        icon={ListChecks}
        label={t("common.select")}
        onClick={onSelect}
      />
      <ContextMenuItem
        icon={ArrowUpDown}
        label={t("common.reorder")}
        onClick={onReorder}
      />

      <div className="h-px bg-white/5 mx-2" />

      <ContextMenuItem
        icon={Trash2}
        label={t("common.remove")}
        onClick={onRemove}
        isWarning
      />
    </motion.div>
  );
};

export default SocialContextMenu;

```

## File: connectfy-client/src/modules/profile/ui/components/SocialLinks/components/SocialLinkPlatformIcons.tsx
```typescript
import { Mail, Globe, Link } from "lucide-react";
import InstagramIcon from "@/assets/icons/InstagramIcon";
import FacebookIcon from "@/assets/icons/FacebookIcon";
import XIcon from "@/assets/icons/XIcon";
import LinkedInIcon from "@/assets/icons/LinkedInIcon";
import YoutubeIcon from "@/assets/icons/YoutubeIcon";
import GithubIcon from "@/assets/icons/GithubIcon";
import RedditIcon from "@/assets/icons/RedditIcon";
import PinterestIcon from "@/assets/icons/PinterestIcon";
import TiktokIcon from "@/assets/icons/TiktokIcon";
import TelegramIcon from "@/assets/icons/TelegramIcon";
import DiscordIcon from "@/assets/icons/DiscordIcon";
import SnapchatIcon from "@/assets/icons/SnapchatIcon";
import WhatsappIcon from "@/assets/icons/WhatsappIcon";
import MediumIcon from "@/assets/icons/MediumIcon";
import BehanceIcon from "@/assets/icons/BehanceIcon";
import { SOCIAL_LINK_PLATFORM } from "@/common/enums/enums";

export const PLATFORM_ICONS: Record<SOCIAL_LINK_PLATFORM, React.ReactNode> = {
  [SOCIAL_LINK_PLATFORM.INSTAGRAM]: <InstagramIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.FACEBOOK]: <FacebookIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.X]: <XIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.LINKEDIN]: <LinkedInIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.YOUTUBE]: <YoutubeIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.GITHUB]: <GithubIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.REDDIT]: <RedditIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.PINTEREST]: <PinterestIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.TIKTOK]: <TiktokIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.TELEGRAM]: <TelegramIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.DISCORD]: <DiscordIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.SNAPCHAT]: <SnapchatIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.WHATSAPP]: <WhatsappIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.MEDIUM]: <MediumIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.BEHANCE]: <BehanceIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.EMAIL]: (
    <Mail size={18} className="text-(--text-color)!" />
  ),
  [SOCIAL_LINK_PLATFORM.WEBSITE]: (
    <Globe size={18} className="text-(--text-color)!" />
  ),
  [SOCIAL_LINK_PLATFORM.OTHER]: (
    <Link size={18} className="text-(--text-color)!" />
  ),
};

```

## File: connectfy-client/src/modules/profile/ui/components/SocialLinks/components/SocialLinkHeader.tsx
```typescript
import { FC, useMemo, Fragment } from "react";
import { useTranslation } from "react-i18next";
import { motion, AnimatePresence } from "framer-motion";
import {
  Link2,
  Plus,
  ListChecks,
  X,
  Trash2,
  ListFilter,
  ArrowUpDown,
  Save,
  MoreVertical,
} from "lucide-react";

import Button from "@/components/ui/CustomButton/Button/Button";
import Dropdown, {
  DropdownOption,
} from "@/components/ui/Select/Dropdown/Dropdown";
import { PrivacyIcon } from "../../PrivacyIcon/PrivacyIcon";
import { PRIVACY_SETTINGS_CHOICE } from "@/common/enums/enums";
import { useIsMobile } from "@/hooks/useIsMobile";
import TextTooltip from "@/components/Tooltip/TextTooltip";

interface ISocialLinkHeaderProps {
  isSelectionMode: boolean;
  isReorderMode: boolean;
  hasLinks: boolean;
  selectedCount: number;
  totalCount: number | undefined;
  activeFilter: "rank" | "newest" | "oldest";
  privacy: PRIVACY_SETTINGS_CHOICE;
  isUpdateRankLoading: boolean;

  onToggleSelectionMode: () => void;
  onToggleReorderMode: () => void;
  onRemoveMultiple: () => void;
  onSaveOrder: () => void;
  onAddClick: () => void;
  onFilterChange: (filter: "rank" | "newest" | "oldest") => void;
}

export const SocialLinkHeader: FC<ISocialLinkHeaderProps> = ({
  isSelectionMode,
  isReorderMode,
  hasLinks,
  selectedCount,
  totalCount,
  activeFilter,
  privacy,
  isUpdateRankLoading,
  onToggleSelectionMode,
  onToggleReorderMode,
  onRemoveMultiple,
  onSaveOrder,
  onAddClick,
  onFilterChange,
}) => {
  const { t } = useTranslation();
  const isMobile = useIsMobile();

  const filterOptions: DropdownOption[] = useMemo(
    () => [
      {
        label: t("common.by_rank"),
        value: "rank",
        isActive: activeFilter === "rank",
        onClick: () => onFilterChange("rank"),
      },
      {
        label: t("common.by_newest"),
        value: "newest",
        isActive: activeFilter === "newest",
        onClick: () => onFilterChange("newest"),
      },
      {
        label: t("common.by_oldest"),
        value: "oldest",
        isActive: activeFilter === "oldest",
        onClick: () => onFilterChange("oldest"),
      },
    ],
    [t, onFilterChange, activeFilter],
  );

  const mobileMoreOptions: DropdownOption[] = useMemo(() => {
    const options: DropdownOption[] = [];
    if (hasLinks) {
      options.push(
        {
          label: t("common.filter"),
          value: "filter",
          icon: <ListFilter size={16} />,
          subMenu: filterOptions,
        },
        {
          label: t("common.reorder"),
          value: "reorder",
          icon: <ArrowUpDown size={16} />,
          onClick: onToggleReorderMode,
        },
        {
          label: t("common.select"),
          value: "select",
          icon: <ListChecks size={16} />,
          onClick: onToggleSelectionMode,
        },
      );
    }
    if (totalCount !== undefined && totalCount < 5) {
      options.push({
        label: t("common.add_link"),
        value: "add",
        icon: <Plus size={16} />,
        onClick: onAddClick,
        className: "!text-(--btn-edit-text) !bg-(--btn-edit-bg)/20 mt-1",
      });
    }
    return options;
  }, [
    hasLinks,
    totalCount,
    filterOptions,
    onToggleReorderMode,
    onToggleSelectionMode,
    onAddClick,
    t,
  ]);

  return (
    <div className="flex flex-row items-center justify-between gap-3 mb-6 md:mb-8 min-h-[40px]">
      <div className="flex items-center gap-2 md:gap-3">
        <Link2 className="w-6 h-6 md:w-7 md:h-7 text-(--primary-color)" />
        <h2 className="m-0 text-lg font-bold md:text-2xl text-(--text-color)">
          {t("common.social_links")}
        </h2>
      </div>

      <div className="flex items-center gap-2 md:gap-3 relative">
        <AnimatePresence mode="wait">
          {isSelectionMode ? (
            <motion.div
              key="selection"
              initial={{ opacity: 0, x: 20 }}
              animate={{ opacity: 1, x: 0 }}
              exit={{ opacity: 0, x: 20 }}
              className="flex items-center gap-2"
            >
              <Button
                className="bg-transparent text-(--text-color) border border-(--auth-glass-border) hover:bg-white/5 font-semibold px-4 py-2 rounded-lg text-sm"
                title={t("common.cancel")}
                icon={<X size={16} />}
                onClick={onToggleSelectionMode}
              />
              <Button
                className="bg-(--error-color) text-white font-semibold px-4 py-2 rounded-lg flex items-center gap-2 hover:opacity-80 border-none text-sm disabled:opacity-50"
                title={`${t("common.remove")} ${selectedCount > 0 ? `(${selectedCount})` : ""}`}
                icon={<Trash2 size={16} />}
                onClick={onRemoveMultiple}
                disabled={selectedCount === 0}
              />
            </motion.div>
          ) : isReorderMode ? (
            <motion.div
              key="reorder"
              initial={{ opacity: 0, x: 20 }}
              animate={{ opacity: 1, x: 0 }}
              exit={{ opacity: 0, x: 20 }}
              className="flex items-center gap-2"
            >
              <Button
                className="bg-transparent text-(--text-color) border border-(--auth-glass-border) hover:bg-white/5 font-semibold px-4 py-2 rounded-lg text-sm"
                title={t("common.cancel")}
                icon={<X size={16} />}
                onClick={onToggleReorderMode}
                disabled={isUpdateRankLoading}
              />
              <Button
                className="bg-(--primary-color) text-white font-semibold px-4 py-2 rounded-lg flex items-center gap-2 hover:opacity-80 border-none text-sm"
                title={t("common.save")}
                icon={<Save size={16} />}
                onClick={onSaveOrder}
                isLoading={isUpdateRankLoading}
              />
            </motion.div>
          ) : (
            <motion.div
              key="default"
              initial={{ opacity: 0, x: -20 }}
              animate={{ opacity: 1, x: 0 }}
              exit={{ opacity: 0, x: -20 }}
              className="flex items-center gap-2 md:gap-3"
            >
              <PrivacyIcon privacy={privacy} fieldName="socialLinks" />
              {isMobile ? (
                <Dropdown
                  options={mobileMoreOptions}
                  icon={<MoreVertical size={20} />}
                  tooltip={t("common.more")}
                />
              ) : (
                <Fragment>
                  <Dropdown
                    options={filterOptions}
                    selected={activeFilter}
                    icon={<ListFilter size={18} />}
                    tooltip={t("common.filter")}
                  />
                  {hasLinks && (
                    <Fragment>
                      <TextTooltip position="top" text={t("common.reorder")}>
                        <Button
                          className="p-1.5 rounded-lg border-none bg-transparent hover:bg-black/5 dark:hover:bg-white/5"
                          icon={<ArrowUpDown size={18} />}
                          onClick={onToggleReorderMode}
                        />
                      </TextTooltip>
                      <TextTooltip position="top" text={t("common.select")}>
                        <Button
                          className="p-1.5 rounded-lg border-none bg-transparent hover:bg-black/5 dark:hover:bg-white/5"
                          icon={<ListChecks size={18} />}
                          onClick={onToggleSelectionMode}
                        />
                      </TextTooltip>
                    </Fragment>
                  )}
                  {totalCount !== undefined && totalCount < 5 && (
                    <Button
                      className="bg-(--btn-edit-bg)! text-(--btn-edit-text)! font-semibold px-4 py-2 rounded-lg border-none text-sm"
                      title={t("common.add_link")}
                      onClick={onAddClick}
                    />
                  )}
                </Fragment>
              )}
            </motion.div>
          )}
        </AnimatePresence>
      </div>
    </div>
  );
};

export default SocialLinkHeader;

```

## File: connectfy-client/src/modules/profile/ui/components/SocialLinks/components/Modal/AddSocialLink.tsx
```typescript
import { FC } from "react";
import { useTranslation } from "react-i18next";
import { useFormik } from "formik";
import { Link2, X, Type, Link } from "lucide-react";
import Modal from "@/components/Modal";
import Button from "@/components/ui/CustomButton/Button/Button";
import Input from "@/components/ui/CustomInput/Input/Input";
import NativeSelect, {
  ISelectOption,
} from "@/components/ui/Select/NativeSelect/NativeSelect";
import { SOCIAL_LINK_PLATFORM } from "@/common/enums/enums";
import { checkEmptyString } from "@/common/utils/checkValues";
import { useErrors } from "@/hooks/useErrors";
import useFormDisabled from "@/hooks/useFormDisabled";
import { useCreateSocialLinkMutation } from "@/modules/profile/api/api";
import { IAddSocialLink } from "@/modules/profile/types/types";
import { snack } from "@/common/utils/snackManager";
import { validateSocialLink } from "@/modules/profile/constants/validation";
import { PLATFORM_ICONS } from "../SocialLinkPlatformIcons";

interface IProps {
  open: boolean;
  onClose: () => void;
  userId: string;
}

const AddSocialLinkModal: FC<IProps> = ({ open, onClose, userId }) => {
  const { t } = useTranslation();
  const { showResponseErrors, showFormikErrors } = useErrors();
  const [create, { isLoading }] = useCreateSocialLinkMutation();

  const platformOptions: ISelectOption[] = Object.values(
    SOCIAL_LINK_PLATFORM,
  ).map((platform) => ({
    label: (
      <div className="flex items-center gap-3">
        <span>{PLATFORM_ICONS[platform]}</span>
        <span>{platform}</span>
      </div>
    ),
    value: platform,
  }));

  const initialState: IAddSocialLink = {
    name: null,
    url: null,
    platform: SOCIAL_LINK_PLATFORM.INSTAGRAM,
    userId,
  };

  const formik = useFormik({
    initialValues: initialState,
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    validate: (values) => validateSocialLink(values, t),
    onSubmit: async (values) => {
      try {
        await create(values).unwrap();
        snack.success(t("user_messages.information_added"));
        handleClose();
      } catch (error) {
        showResponseErrors(error);
      }
    },
  });

  const isDisabled = useFormDisabled<IAddSocialLink>({
    formik,
    loading: isLoading,
    validationRules: [
      (values) => {
        const requiredStringFields = [values.name, values.url, values.platform];
        return requiredStringFields.every((f) => f && checkEmptyString(f));
      },
    ],
  });

  const handleClose = () => {
    formik.resetForm();
    onClose();
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    const errors = await formik.validateForm();
    if (Object.keys(errors).length > 0) {
      showFormikErrors(errors);
      return;
    }
    formik.handleSubmit(e as any);
  };

  return (
    <Modal open={open} onClose={handleClose}>
      <div className="w-full max-w-[550px] bg-(--auth-main-bg)! rounded-2xl shadow-(--card-shadow) overflow-hidden animate-slide-up duration-600">
        {/* Modal Header - PersonalInfoModal ilə eyni */}
        <div className="p-6 border-b border-(--auth-glass-border) relative">
          <h2 className="text-xl font-bold text-(--text-primary) flex items-center gap-2">
            <Link2 size={22} className="text-(--primary-color)" />
            {t("common.add_social_link")}
          </h2>
          <p className="text-sm text-(--muted-color) mt-1">
            {t("common.add_social_link_subtitle")}
          </p>

          {/* Bağla düyməsi */}
          <Button
            onClick={handleClose}
            className="absolute top-6 right-6 p-1 rounded-lg hover:bg-black/5 dark:hover:bg-white/5 text-(--muted-color) transition-colors"
            icon={<X size={20} />}
          />
        </div>

        {/* Modal Body */}
        <form onSubmit={handleSubmit} className="p-6">
          <div className="flex flex-col gap-6">
            {/* Platform Seçimi */}
            <NativeSelect
              title={t("common.platform")}
              inputSize="large"
              options={platformOptions}
              value={formik.values.platform}
              onChange={(val) => formik.setFieldValue("platform", val)}
              disabled={isLoading}
            />

            {/* Link Adı */}
            <Input
              name="name"
              title={t("common.name")}
              isFloating
              icon={<Type size={18} />}
              value={formik.values.name || ""}
              onChange={formik.handleChange}
              maxLength={50}
              isError={!!formik.errors.name}
            />

            {/* URL/Link */}
            <Input
              name="url"
              title={t("common.link")}
              isFloating
              icon={<Link size={18} />}
              value={formik.values.url || ""}
              onChange={formik.handleChange}
              maxLength={200}
              isError={!!formik.errors.url}
            />
          </div>

          {/* Action Buttons - PersonalInfoModal ilə eyni */}
          <div className="flex items-center gap-3 mt-8">
            <Button
              type="button"
              className="flex-1 h-12 rounded-xl text-sm font-bold bg-black/5 dark:bg-white/5 text-(--text-primary) hover:bg-black/10 dark:hover:bg-white/10 transition-all border-none"
              onClick={handleClose}
              disabled={isLoading}
              title={t("common.cancel")}
            />
            <Button
              type="submit"
              className="flex-1 h-12 rounded-xl text-sm font-bold bg-linear-to-r from-(--primary-color) to-(--third-color) text-white shadow-lg shadow-(--primary-color)/20 transition-all duration-600 border-none"
              disabled={isDisabled}
              isLoading={isLoading}
              title={t("common.save")}
            />
          </div>
        </form>
      </div>
    </Modal>
  );
};

export default AddSocialLinkModal;

```

## File: connectfy-client/src/modules/profile/ui/components/SocialLinks/components/Modal/EditSocialLink.tsx
```typescript
import { FC } from "react";
import { useTranslation } from "react-i18next";
import { useFormik } from "formik";
import { Link2, X, Link, Type } from "lucide-react";
import Modal from "@/components/Modal";
import Button from "@/components/ui/CustomButton/Button/Button";
import Input from "@/components/ui/CustomInput/Input/Input";
import NativeSelect, {
  ISelectOption,
} from "@/components/ui/Select/NativeSelect/NativeSelect";
import { SOCIAL_LINK_PLATFORM } from "@/common/enums/enums";
import { checkEmptyString } from "@/common/utils/checkValues";
import { useErrors } from "@/hooks/useErrors";
import useFormDisabled from "@/hooks/useFormDisabled";
import { useUpdateSocialLinkMutation } from "@/modules/profile/api/api";
import { IEditSocialLink, ISocialLink } from "@/modules/profile/types/types";
import { snack } from "@/common/utils/snackManager";
import { getChangedData } from "@/common/utils/getDirtyValues";
import { validateSocialLink } from "@/modules/profile/constants/validation";
import { PLATFORM_ICONS } from "../SocialLinkPlatformIcons";

interface IProps {
  open: boolean;
  onClose: () => void;
  socialLink: ISocialLink;
}

const EditSocialLinkModal: FC<IProps> = ({ open, onClose, socialLink }) => {
  const { t } = useTranslation();
  const { showResponseErrors, showFormikErrors } = useErrors();
  const [update, { isLoading }] = useUpdateSocialLinkMutation();

  const platformOptions: ISelectOption[] = Object.values(
    SOCIAL_LINK_PLATFORM,
  ).map((platform) => ({
    label: (
      <div className="flex items-center gap-3">
        <span>{PLATFORM_ICONS[platform]}</span>
        <span>{platform}</span>
      </div>
    ),
    value: platform,
  }));

  const initialState: IEditSocialLink = {
    _id: socialLink._id,
    name: socialLink.name,
    url: socialLink.url,
    platform: socialLink.platform,
  };

  const formik = useFormik({
    initialValues: initialState,
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    validate: (values) => validateSocialLink(values, t),
    onSubmit: async (values) => {
      try {
        const changedData = getChangedData<IEditSocialLink>(socialLink, values);
        await update({ ...changedData, _id: socialLink._id }).unwrap();
        snack.success(t("user_messages.information_updated"));
        handleClose();
      } catch (error) {
        showResponseErrors(error);
      }
    },
  });

  const isDisabled = useFormDisabled<IEditSocialLink>({
    formik,
    loading: isLoading,
    validationRules: [
      (values) => {
        const requiredStringFields = [values.name, values.url, values.platform];
        return requiredStringFields.every((f) => f && checkEmptyString(f));
      },
      formik.dirty,
    ],
  });

  const handleClose = () => {
    formik.resetForm();
    onClose();
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    const errors = await formik.validateForm();
    if (Object.keys(errors).length > 0) {
      showFormikErrors(errors);
      return;
    }
    formik.handleSubmit(e as any);
  };

  return (
    <Modal open={open} onClose={handleClose}>
      <div className="w-full max-w-[550px] bg-(--auth-main-bg)! rounded-2xl shadow-(--card-shadow) overflow-hidden animate-slide-up duration-600">
        {/* Modal Header - PersonalInfoModal ilə eyni */}
        <div className="p-6 border-b border-(--auth-glass-border) relative">
          <h2 className="text-xl font-bold text-(--text-primary) flex items-center gap-2">
            <Link2 size={22} className="text-(--primary-color)" />
            {t("common.edit_social_link")}
          </h2>
          <p className="text-sm text-(--muted-color) mt-1">
            {t("common.edit_social_link_subtitle")}
          </p>

          {/* Bağla düyməsi */}
          <Button
            onClick={handleClose}
            className="absolute top-6 right-6 p-1 rounded-lg hover:bg-black/5 dark:hover:bg-white/5 text-(--muted-color) transition-colors"
            icon={<X size={20} />}
          />
        </div>

        {/* Modal Body */}
        <form onSubmit={handleSubmit} className="p-6">
          <div className="flex flex-col gap-6">
            {/* Platform Seçimi */}
            <NativeSelect
              title={t("common.platform")}
              inputSize="large"
              // icon={
              //   PLATFORM_ICONS[formik.values.platform as SOCIAL_LINK_PLATFORM]
              // }
              options={platformOptions}
              value={formik.values.platform ?? SOCIAL_LINK_PLATFORM.INSTAGRAM}
              onChange={(val) => formik.setFieldValue("platform", val)}
              disabled={isLoading}
            />

            {/* Link Adı */}
            <Input
              name="name"
              title={t("common.name")}
              isFloating
              icon={<Type size={18} />}
              value={formik.values.name || ""}
              onChange={formik.handleChange}
              maxLength={50}
              isError={!!formik.errors.name}
            />

            {/* URL/Link */}
            <Input
              name="url"
              title={t("common.link")}
              isFloating
              icon={<Link size={18} />}
              value={formik.values.url || ""}
              onChange={formik.handleChange}
              maxLength={200}
              isError={!!formik.errors.url}
            />
          </div>

          {/* Action Buttons - PersonalInfoModal ilə eyni */}
          <div className="flex items-center gap-3 mt-8">
            <Button
              type="button"
              className="flex-1 h-12 rounded-xl text-sm font-bold bg-black/5 dark:bg-white/5 text-(--text-primary) hover:bg-black/10 dark:hover:bg-white/10 transition-all border-none"
              onClick={handleClose}
              disabled={isLoading}
              title={t("common.cancel")}
            />
            <Button
              type="submit"
              className="flex-1 h-12 rounded-xl text-sm font-bold bg-linear-to-r from-(--primary-color) to-(--third-color) text-white shadow-lg shadow-(--primary-color)/20 transition-all duration-600 border-none"
              disabled={isDisabled}
              isLoading={isLoading}
              title={t("common.save")}
            />
          </div>
        </form>
      </div>
    </Modal>
  );
};

export default EditSocialLinkModal;

```

## File: connectfy-client/src/modules/profile/router/router.tsx
```typescript
import { RouteObject } from "react-router-dom";
import { lazy } from "react";
import { ROUTER } from "@/common/constants/routet";
import ComponentLoader from "@/components/Loader/Components/ComponentLoader";
import MainCardSkeleton from "@/components/Skeleton/profile/MainCardSkeleton";
import PersonalInformationSkeleton from "@/components/Skeleton/profile/PersonalInformationSkeleton";
import BioSkeleton from "@/components/Skeleton/profile/BioSkeleton";
import ProfileHeader from "../ui/components/ProfileHeader/ProfileHeader";

const ProfilePageSkeleton = () => (
  <div className="relative w-full h-screen overflow-x-hidden overflow-y-auto font-sans scroll-smooth bg-(--bg-color)">
    <ProfileHeader />

    <main className="mx-auto max-w-[900px] pt-8 px-6 pb-[60px]">
      <MainCardSkeleton />
      <PersonalInformationSkeleton />
      <BioSkeleton />
    </main>
  </div>
);

const Profile = ComponentLoader(
  lazy(() => import("../ui/Profile")),
  <ProfilePageSkeleton />,
);

const routes: RouteObject[] = [
  {
    path: ROUTER.PROFILE.MAIN,
    element: <Profile />,
  },
];

export default routes;

```

## File: connectfy-client/src/modules/profile/types/types.ts
```typescript
import {
  AvatarFormats,
  GENDER,
  LANGUAGE,
  ProfilePhotoUpdateAction,
  PROVIDER,
  SOCIAL_LINK_PLATFORM,
} from "@/common/enums/enums";
import { IPhoneNumber } from "@/modules/auth/types/types";

export interface IAccount {
  _id: string;
  userId: string;
  firstName: string;
  lastName: string;
  fullName: string;
  gender: GENDER;
  bio: string | null;
  location: string | null;
  avatar: IAvatar | null;
  defaultAvatar: IDefaultAvatar;
  lastSeen: Date;
  birthdayDate: Date;
}

export interface IAvatar {
  key: string | null;
  url: string;
  isCustom: boolean;
}

export interface IUser {
  _id: string;
  username: string;
  email: string;
  phoneNumber: IPhoneNumber;
  isTwoFactorEnabled: boolean;
  timeZone: string | null;
  location: string | null;
  provider: PROVIDER;
  createdAt: Date;
  updatedAt: Date;
}

export interface IMe extends IUser {
  avatar: IAvatar | null;
  defaultAvatar: IDefaultAvatar;
  language: LANGUAGE;
}

export interface IDefaultAvatar {
  format: AvatarFormats;
  seed: string;
  url: string;
}

export interface IEditProfile extends Partial<
  Omit<IAccount, "_id" | "userId" | "lastSeen">
> {
  _id: string;
}

export interface IEditAvatar {
  _id: string;
  action: ProfilePhotoUpdateAction;
  avatar: IAvatar | null;
}

export interface IEditDefaultAvatar {
  _id: string;
  format: AvatarFormats;
  useDefaultAvatar: boolean;
}

export interface IFindSocialLinks {
  userId: string;
  sort?: Record<string, 1 | -1>;
}

export interface ISocialLink {
  _id: string;
  userId: string;
  name: string;
  rank: number;
  url: string;
  platform: SOCIAL_LINK_PLATFORM;
  createdAt: Date;
  updatedAt: Date;
}

export interface IAddSocialLink {
  name: string | null;
  url: string | null;
  platform: SOCIAL_LINK_PLATFORM;
  userId: string;
}

export interface IEditSocialLink extends Partial<
  Omit<ISocialLink, "_id" | "userId">
> {
  _id: string;
}

export interface IUpdateSocialLinkRank {
  links: {
    _id: string;
    rank: number;
  }[];
  userId: string;
}

export interface IRemoveSocialLink {
  _id: string;
}

export interface IRemoveAllSocialLinks {
  _ids: string[];
  userId: string;
}

```

## File: connectfy-client/src/modules/profile/api/api.ts
```typescript
import { baseQuery } from "@/common/api/axiosBaseQuery";
import { RESOURCE } from "@/common/enums/enums";
import { createApi } from "@reduxjs/toolkit/query/react";
import {
  IAccount,
  IAddSocialLink,
  IEditAvatar,
  IEditDefaultAvatar,
  IEditProfile,
  IEditSocialLink,
  IFindSocialLinks,
  IMe,
  IRemoveAllSocialLinks,
  IRemoveSocialLink,
  IUpdateSocialLinkRank,
} from "../types/types";
import { API_ENDPOINTS } from "@/common/constants/apiEndpoints";
import {
  IFindAllResponse,
  IRemoveAllResponse,
  IRemoveResponse,
  IUpdateResponse,
} from "@/common/interfaces/interfaces";
import { ISocialLink } from "../types/types";

export const profileApi = createApi({
  reducerPath: RESOURCE.PROFILE,
  baseQuery: baseQuery,
  tagTypes: ["User", "Account", "SocialLink"],
  endpoints: (builder) => ({
    // <====================== PROFILE ======================>
    // <====================== PROFILE ======================>
    // ====================== GET ME
    getMe: builder.query<IMe, void>({
      query: () => ({
        url: API_ENDPOINTS.USER.ME,
        method: "POST",
      }),
      providesTags: (result) =>
        result && result?._id
          ? [{ type: "User", id: result._id }]
          : [{ type: "User", id: "LIST" }],
    }),

    // ====================== GET ACCOUNT
    getAccount: builder.query<IAccount, void>({
      query: () => ({
        url: API_ENDPOINTS.ACCOUNT.PROFILE.GET,
        method: "POST",
      }),
      providesTags: ["Account"],
    }),

    // ====================== UPDATE PROFILE
    updateProfile: builder.mutation<IUpdateResponse, IEditProfile>({
      query: (body) => ({
        url: API_ENDPOINTS.ACCOUNT.PROFILE.UPDATE,
        method: "PATCH",
        body,
      }),
      onQueryStarted: async (arg, { dispatch, queryFulfilled }) => {
        const { _id, ...updatedFields } = arg;

        const patchResult = dispatch(
          profileApi.util.updateQueryData("getAccount", undefined, (draft) => {
            if (draft) {
              Object.assign(draft, updatedFields);
            }
          }),
        );

        try {
          await queryFulfilled;
        } catch (err) {
          patchResult.undo();
        }
      },
    }),

    // ====================== UPDATE AVATAR
    updateAvatar: builder.mutation<IUpdateResponse, IEditAvatar>({
      query: (body) => ({
        url: API_ENDPOINTS.ACCOUNT.PROFILE.UPDATE_AVATAR,
        method: "PATCH",
        body,
      }),
      onQueryStarted: async (_, { dispatch, queryFulfilled }) => {
        const { data } = await queryFulfilled;

        const patchResult = dispatch(
          profileApi.util.updateQueryData("getAccount", undefined, (draft) => {
            if (draft) {
              draft.avatar = data.avatar;
            }
          }),
        );

        const patchResultForGetMe = dispatch(
          profileApi.util.updateQueryData("getMe", undefined, (draft) => {
            if (draft) {
              draft.avatar = data.avatar;
            }
          }),
        );

        try {
          await queryFulfilled;
        } catch (err) {
          patchResult.undo();
          patchResultForGetMe.undo();
        }
      },
    }),

    // ====================== UPDATE DEFAULT AVATAR
    updateDefaultAvatar: builder.mutation<IUpdateResponse, IEditDefaultAvatar>({
      query: (body) => ({
        url: API_ENDPOINTS.ACCOUNT.PROFILE.UPDATE_DEFAULT_AVATAR,
        method: "PATCH",
        body,
      }),
      onQueryStarted: async (_, { dispatch, queryFulfilled }) => {
        const { data } = await queryFulfilled;

        const patchResult = dispatch(
          profileApi.util.updateQueryData("getAccount", undefined, (draft) => {
            if (draft) {
              draft.defaultAvatar = data.defaultAvatar;
              if (data.avatar) {
                draft.avatar = data.avatar;
              }
            }
          }),
        );

        const patchResultForGetMe = dispatch(
          profileApi.util.updateQueryData("getMe", undefined, (draft) => {
            if (draft) {
              draft.defaultAvatar = data.defaultAvatar;
              if (data.avatar) {
                draft.avatar = data.avatar;
              }
            }
          }),
        );

        try {
          await queryFulfilled;
        } catch (err) {
          patchResult.undo();
          patchResultForGetMe.undo();
        }
      },
    }),
    // <====================== PROFILE ======================>
    // <====================== PROFILE ======================>

    // <====================== SOCIAL LINKS ======================>
    // <====================== SOCIAL LINKS ======================>
    // ====================== GET SOCIAL LINKS
    /**
     * Social linkləri gətirir.
     * providesTags: Bu query-nin nəticəsini müəyyən tag-lərlə nişanlayır.
     */
    getSocialLinks: builder.query<
      IFindAllResponse<ISocialLink>,
      IFindSocialLinks
    >({
      query: (body) => ({
        url: API_ENDPOINTS.ACCOUNT.SOCIAL_LINK.GET,
        method: "POST",
        body,
      }),
      providesTags: (result) =>
        result
          ? [
              ...result.data.map(({ _id }) => ({
                type: "SocialLink" as const,
                id: _id,
              })),
              { type: "SocialLink", id: "LIST" },
            ]
          : [{ type: "SocialLink", id: "LIST" }],
    }),

    createSocialLink: builder.mutation<ISocialLink, IAddSocialLink>({
      query: (body) => ({
        url: API_ENDPOINTS.ACCOUNT.SOCIAL_LINK.CREATE,
        method: "POST",
        body,
      }),
      invalidatesTags: [{ type: "SocialLink", id: "LIST" }],
    }),

    updateSocialLink: builder.mutation<IUpdateResponse, IEditSocialLink>({
      query: (body) => ({
        url: API_ENDPOINTS.ACCOUNT.SOCIAL_LINK.UPDATE,
        method: "PATCH",
        body,
      }),
      invalidatesTags: (_, __, { _id }) => [
        { type: "SocialLink", id: _id },
        { type: "SocialLink", id: "LIST" },
      ],
    }),

    updateSocialLinkRank: builder.mutation<
      IUpdateResponse,
      IUpdateSocialLinkRank
    >({
      query: (body) => ({
        url: API_ENDPOINTS.ACCOUNT.SOCIAL_LINK.UPDATE_RANK,
        method: "PATCH",
        body,
      }),
      async onQueryStarted({ userId, links }, { dispatch, queryFulfilled }) {
        const patchResult = dispatch(
          profileApi.util.updateQueryData(
            "getSocialLinks",
            { userId },
            (draft) => {
              if (draft?.data) {
                links.forEach((updatedLink) => {
                  const item = draft.data.find(
                    (l) => l._id === updatedLink._id,
                  );
                  if (item) {
                    item.rank = updatedLink.rank;
                  }
                });
              }
            },
          ),
        );

        try {
          await queryFulfilled;
        } catch {
          patchResult.undo();
        }
      },
    }),

    removeSocialLink: builder.mutation<IRemoveResponse, IRemoveSocialLink>({
      query: (body) => ({
        url: API_ENDPOINTS.ACCOUNT.SOCIAL_LINK.REMOVE,
        method: "DELETE",
        body,
      }),
      invalidatesTags: (_, __, { _id }) => [
        { type: "SocialLink", id: _id },
        { type: "SocialLink", id: "LIST" },
      ],
    }),

    removeSocialLinks: builder.mutation<
      IRemoveAllResponse,
      IRemoveAllSocialLinks
    >({
      query: ({ _ids }) => ({
        url: API_ENDPOINTS.ACCOUNT.SOCIAL_LINK.REMOVE_MANY,
        method: "DELETE",
        body: { _ids },
      }),
      invalidatesTags: [{ type: "SocialLink", id: "LIST" }],
    }),
    // <====================== SOCIAL LINKS ======================>
    // <====================== SOCIAL LINKS ======================>
  }),
});

export const {
  useGetMeQuery,
  useGetAccountQuery,
  useUpdateProfileMutation,
  useUpdateAvatarMutation,
  useUpdateDefaultAvatarMutation,
  useGetSocialLinksQuery,
  useCreateSocialLinkMutation,
  useUpdateSocialLinkMutation,
  useUpdateSocialLinkRankMutation,
  useRemoveSocialLinkMutation,
  useRemoveSocialLinksMutation,
} = profileApi;

```

## File: connectfy-client/src/modules/settings/BackgroundSettings/ui/BackgroundSettings.tsx
```typescript
import "./backgroundSettings.style.css";

const BackgroundSettings = () => {
  return <div>BackgroundSettingsPage</div>;
};

export default BackgroundSettings;

```

## File: connectfy-client/src/modules/settings/BackgroundSettings/router/router.tsx
```typescript
import { RouteObject } from "react-router-dom";
import { lazy } from "react";
import { ROUTER } from "@/common/constants/routet";
import ComponentLoader from "@/components/Loader/Components/ComponentLoader";

const BackgroundSettings = ComponentLoader(
  lazy(() => import("../ui/BackgroundSettings")),
);

const routes: RouteObject[] = [
  {
    path: ROUTER.SETTINGS.BACKGROUND,
    element: <BackgroundSettings />,
  },
];

export default routes;

```

## File: connectfy-client/src/modules/settings/PrivacySettings/hooks/usePrivacySettings.tsx
```typescript
import { useAuthStore } from "@/store/zustand/useAuthStore";
import { useUser } from "@/context/UserContext";
import { useGetPrivacySettingsQuery } from "@/modules/settings/PrivacySettings/api/api.ts";

export function usePrivacySettings() {
  const { access_token } = useAuthStore();
  const { isSuccess, isError } = useUser();

  const result = useGetPrivacySettingsQuery(undefined, {
    skip: !access_token || !isSuccess || isError,
  });

  return {
    privacySettings: result.data,
    ...result,
  };
}

```

## File: connectfy-client/src/modules/settings/PrivacySettings/constants/constant.ts
```typescript
import { PRIVACY_SETTINGS_CHOICE } from "@/common/enums/enums";
import { TFunction } from "i18next";
import { Eye, UserCheck, Lock } from "lucide-react";
import { IEditPrivacySettings, IPrivacySettings } from "../types/types";
import { snack } from "@/common/utils/snackManager";

export const privacyOptions = (t: TFunction) => {
  return [
    {
      key: PRIVACY_SETTINGS_CHOICE.EVERYONE,
      name: t(`enum.${PRIVACY_SETTINGS_CHOICE.EVERYONE}`),
      icon: Eye,
    },
    {
      key: PRIVACY_SETTINGS_CHOICE.MY_FRIENDS,
      name: t(`enum.${PRIVACY_SETTINGS_CHOICE.MY_FRIENDS}`),
      icon: UserCheck,
    },
    {
      key: PRIVACY_SETTINGS_CHOICE.NOBODY,
      name: t(`enum.${PRIVACY_SETTINGS_CHOICE.NOBODY}`),
      icon: Lock,
    },
  ];
};

export const PRIVACY_FIELDS = (t: TFunction) => [
  { name: "email", label: t("common.email_visibility"), field: "email" },
  { name: "gender", label: t("common.gender_visibility"), field: "gender" },
  { name: "avatar", label: t("common.avatar_visibility"), field: "avatar" },
  {
    name: "birthdayDate",
    label: t("common.birthday_date_visibility"),
    field: "birthdayDate",
  },
  { name: "bio", label: t("common.bio_visibility"), field: "bio" },
  {
    name: "location",
    label: t("common.location_visibility"),
    field: "location",
  },
  {
    name: "socialLinks",
    label: t("common.social_visibility"),
    field: "socialLinks",
  },
  {
    name: "lastSeen",
    label: t("common.last_seen_visibility"),
    field: "lastSeen",
  },
  {
    name: "phoneNumber",
    label: t("common.phone_visibility"),
    field: "phoneNumber",
  },
];

export const initialState = (data?: IPrivacySettings): IEditPrivacySettings => {
  let initialSale: IEditPrivacySettings;

  if (data) {
    const {
      _id,
      email,
      bio,
      gender,
      location,
      socialLinks,
      lastSeen,
      avatar,
      messageRequest,
      birthdayDate,
      friendshipRequest,
      readReceipts,
      phoneNumber,
    } = data;

    initialSale = {
      _id,
      email,
      bio,
      gender,
      location,
      socialLinks,
      lastSeen,
      avatar,
      messageRequest,
      birthdayDate,
      friendshipRequest,
      readReceipts,
      phoneNumber,
    };
  } else {
    initialSale = {
      _id: undefined,
      email: undefined,
      bio: undefined,
      gender: undefined,
      location: undefined,
      socialLinks: undefined,
      lastSeen: undefined,
      avatar: undefined,
      messageRequest: undefined,
      birthdayDate: undefined,
      friendshipRequest: undefined,
      readReceipts: undefined,
      phoneNumber: undefined,
    };
  }

  return initialSale;
};

export const validatePrivacySettings = (
  values: IEditPrivacySettings,
  t: TFunction,
): void => {
  const {
    email,
    bio,
    gender,
    location,
    socialLinks,
    lastSeen,
    avatar,
    messageRequest,
    birthdayDate,
    phoneNumber,
  } = values;

  const valuesArray = [
    email,
    bio,
    gender,
    location,
    socialLinks,
    lastSeen,
    avatar,
    messageRequest,
    birthdayDate,
    phoneNumber,
  ];

  valuesArray.forEach((val) => {
    if (val && !Object.values(PRIVACY_SETTINGS_CHOICE).includes(val)) {
      snack.error(t("error_messages.invalid_choice"));
      return;
    }
  });
};

```

## File: connectfy-client/src/modules/settings/PrivacySettings/ui/PrivacySettings.tsx
```typescript
import "./privacySettings.style.css";
import { CheckCircle, Shield, UserCheck } from "lucide-react";
import { Fragment } from "react";
import CustomSelect from "@/components/ui/Select/CustomSelect/CustomSelect";
import { useTranslation } from "react-i18next";
import { PRIVACY_SETTINGS_CHOICE } from "@/common/enums/enums";
import UniqueHeader from "@/components/Header/UnqiueHeader/UniqueHeader";
import SettingCard from "@/components/Card/SettingsCard/SettingCard";
import ToggleCard from "@/components/Card/ToggleCard/ToggleCard";
import { ROUTER } from "@/common/constants/routet";
import {
  initialState,
  PRIVACY_FIELDS,
  privacyOptions,
  validatePrivacySettings,
} from "../constants/constant";
import { useFormik } from "formik";
import { snack } from "@/common/utils/snackManager";
import { useBlocker } from "@/hooks/useBlocker";
import { useBeforeUnload } from "@/hooks/useBeforeUnload";
import SaveChangesModal from "@/components/Modal/SaveChangesModal/SaveChangesModal";
import { useEditPrivacySettingsMutation } from "../api/api";
import { useErrors } from "@/hooks/useErrors";
import { SettingsSkeleton } from "@/common/utils/skeleton";
import { getChangedData } from "@/common/utils/getDirtyValues";
import { IEditPrivacySettings } from "../types/types";
import { usePrivacySettings } from "../hooks/usePrivacySettings";
import { useAppNavigation } from "@/hooks/useAppNavigation";

const PrivacySettings = () => {
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();
  const { showResponseErrors } = useErrors();

  const { privacySettings, isLoading: LOADING_GET } = usePrivacySettings();

  const [editPrivacySettings, { isLoading: LOADING_UPDATE }] =
    useEditPrivacySettingsMutation();

  const formik = useFormik({
    initialValues: initialState(privacySettings!),
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    validate: (values) => validatePrivacySettings(values, t),
    onSubmit: async (values, { resetForm }) => {
      try {
        const changedData = getChangedData<IEditPrivacySettings>(
          privacySettings!,
          values,
        );
        values = {
          _id: privacySettings!._id,
          ...changedData,
        };
        await editPrivacySettings(values).unwrap();
        snack.success(t("user_messages.information_updated"));
        resetForm();
      } catch (error) {
        showResponseErrors(error);
      }
    },
  });

  const { pending, confirm, cancel } = useBlocker(!!formik.dirty);
  useBeforeUnload(!!formik.dirty, t("common.unsaved_changes_message"));

  const PRIVACY_OPTIONS = privacyOptions(t);

  const getLabel = (key?: string | null) =>
    PRIVACY_OPTIONS.find((opt) => opt.key === key)?.name || t("common.select");

  const createOptions = (
    activeKey: string,
    onSelect: (value: PRIVACY_SETTINGS_CHOICE) => void,
  ) => ({
    title: t("common.privacy_level"),
    activeKey,
    selections: PRIVACY_OPTIONS.map((opt) => ({
      key: opt.key,
      name: opt.name,
      title: opt.name,
      icon: opt.icon,
      onClick: () => onSelect(opt.key as PRIVACY_SETTINGS_CHOICE),
    })),
  });

  const onClickBack = () => navigate(ROUTER.SETTINGS.MAIN);

  const privacyFields = PRIVACY_FIELDS(t);

  const handleSaveAndLeave = async () => {
    await formik.submitForm();
    confirm();
  };

  const handleDiscardChanges = () => {
    formik.resetForm();
    confirm();
  };

  const handleCancelModal = () => {
    cancel();
  };

  return (
    <Fragment>
      <section className="privacy-settings">
        <UniqueHeader
          headerTitle={t("common.privacy_header")}
          headerSubtitle={t("common.privacy_description")}
          onClickBack={onClickBack}
          onHeaderButtonClick={formik.handleSubmit}
          showHeaderButton
          isHeaderButtonDisabled={!formik.dirty || LOADING_UPDATE}
          isLoading={LOADING_UPDATE}
        />
        <div className="privacy-settings-container">
          {LOADING_GET ? (
            <SettingsSkeleton count={4} />
          ) : (
            <Fragment>
              <div className="privacy-settings-content">
                {/* ACTION PRIVACY */}
                <div className="privacy-section">
                  <h2 className="privacy-section-title">
                    {t("common.activity_privacy")}
                  </h2>

                  <ToggleCard
                    header={{
                      icon: CheckCircle,
                      title: t("common.read_receipts"),
                      subtitle: t("common.read_receipts_desc"),
                    }}
                    slider={{
                      checked: !!formik.values.readReceipts,
                      onClick: () =>
                        formik.setFieldValue(
                          "readReceipts",
                          !formik.values.readReceipts,
                        ),
                    }}
                  />

                  <ToggleCard
                    header={{
                      icon: UserCheck,
                      title: t("common.friend_requests"),
                      subtitle: t("common.friend_requests_desc"),
                    }}
                    slider={{
                      checked: !!formik.values.friendshipRequest,
                      onClick: () =>
                        formik.setFieldValue(
                          "friendshipRequest",
                          !formik.values.friendshipRequest,
                        ),
                    }}
                  />

                  <SettingCard
                    header={{
                      icon: Shield,
                      title: t("common.message_privacy"),
                      subtitle: t("common.message_privacy_desc"),
                    }}
                  >
                    <CustomSelect
                      buttonTitle={getLabel(
                        formik.values.messageRequest as PRIVACY_SETTINGS_CHOICE,
                      )}
                      options={createOptions(
                        formik.values.messageRequest as PRIVACY_SETTINGS_CHOICE,
                        (val) => formik.setFieldValue("messageRequest", val),
                      )}
                    />
                  </SettingCard>
                </div>

                {/* DATA PRIVACY */}
                <div className="privacy-section">
                  <h2 className="privacy-section-title">
                    {t("common.data_privacy")}
                  </h2>

                  <SettingCard
                    cardStyle={{
                      padding: "5px 16px 20px 16px",
                    }}
                  >
                    <Fragment>
                      {privacyFields.map((item, idx) => {
                        const activeValue = formik.values[
                          item.field as keyof typeof formik.values
                        ] as string | undefined;
                        return (
                          <div
                            className="privacy-row"
                            key={item.field}
                            style={idx > 0 ? { marginTop: "12px" } : {}}
                          >
                            <span className="privacy-row-label">
                              {item.label}
                            </span>
                            <div className="privacy-row-select">
                              <CustomSelect
                                buttonTitle={getLabel(activeValue)}
                                options={createOptions(
                                  activeValue as string,
                                  (val) =>
                                    formik.setFieldValue(item.field, val),
                                )}
                              />
                            </div>
                          </div>
                        );
                      })}
                    </Fragment>
                  </SettingCard>
                </div>
              </div>
            </Fragment>
          )}
        </div>
      </section>

      <SaveChangesModal
        open={!!pending}
        handleSave={handleSaveAndLeave}
        handleCancel={handleCancelModal}
        handleDiscardChanges={handleDiscardChanges}
        isLoading={LOADING_UPDATE}
      />
    </Fragment>
  );
};

export default PrivacySettings;

```

## File: connectfy-client/src/modules/settings/PrivacySettings/router/router.tsx
```typescript
import { RouteObject } from "react-router-dom";
import { lazy } from "react";
import { ROUTER } from "@/common/constants/routet";
import ComponentLoader from "@/components/Loader/Components/ComponentLoader";

const PrivacySettings = ComponentLoader(
  lazy(() => import("../ui/PrivacySettings")),
);

const routes: RouteObject[] = [
  {
    path: ROUTER.SETTINGS.PRIVACY,
    element: <PrivacySettings />,
  },
];

export default routes;

```

## File: connectfy-client/src/modules/settings/PrivacySettings/types/types.ts
```typescript
import { PRIVACY_SETTINGS_CHOICE } from "@/common/enums/enums";

export interface IPrivacySettings {
  _id: string;
  userId: string;
  email: PRIVACY_SETTINGS_CHOICE;
  bio: PRIVACY_SETTINGS_CHOICE;
  gender: PRIVACY_SETTINGS_CHOICE;
  location: PRIVACY_SETTINGS_CHOICE;
  socialLinks: PRIVACY_SETTINGS_CHOICE;
  lastSeen: PRIVACY_SETTINGS_CHOICE;
  avatar: PRIVACY_SETTINGS_CHOICE;
  messageRequest: PRIVACY_SETTINGS_CHOICE;
  birthdayDate: PRIVACY_SETTINGS_CHOICE;
  phoneNumber: PRIVACY_SETTINGS_CHOICE;
  friendshipRequest: boolean;
  readReceipts: boolean;
}

export interface IEditPrivacySettings
  extends Partial<Omit<IPrivacySettings, "userId">> {}

```

## File: connectfy-client/src/modules/settings/PrivacySettings/api/api.ts
```typescript
import { baseQuery } from "@/common/api/axiosBaseQuery";
import { RESOURCE } from "@/common/enums/enums";
import { createApi } from "@reduxjs/toolkit/query/react";
import { API_ENDPOINTS } from "@/common/constants/apiEndpoints";
import { IEditPrivacySettings, IPrivacySettings } from "../types/types";
import { IUpdateResponse } from "@/common/interfaces/interfaces";

export const privacySettingsApi = createApi({
  reducerPath: RESOURCE.PRIVACY_SETTINGS,
  baseQuery: baseQuery,
  tagTypes: ["PrivacySettings"],
  endpoints: (builder) => ({
    // ====================== GET PRIVACY SETTINGS
    getPrivacySettings: builder.query<IPrivacySettings, void>({
      query: () => ({
        url: API_ENDPOINTS.ACCOUNT.SETTINGS.PRIVACY_SETTINGS.GET,
        method: "POST",
      }),
      providesTags: ["PrivacySettings"],
    }),

    // ====================== EDIT PRIVACY SETTINGS
    editPrivacySettings: builder.mutation<
      IUpdateResponse,
      IEditPrivacySettings
    >({
      query: (data) => ({
        url: API_ENDPOINTS.ACCOUNT.SETTINGS.PRIVACY_SETTINGS.UPDATE,
        method: "PATCH",
        body: data,
      }),

      onQueryStarted: async (arg, { dispatch, queryFulfilled }) => {
        const { _id, ...updatedFields } = arg;

        const patchResult = dispatch(
          privacySettingsApi.util.updateQueryData(
            "getPrivacySettings",
            undefined,
            (draft) => {
              if (draft) {
                Object.assign(draft, updatedFields);
              }
            },
          ),
        );

        try {
          await queryFulfilled;
        } catch (error) {
          patchResult.undo();
        }
      },
    }),
  }),
});

export const { useGetPrivacySettingsQuery, useEditPrivacySettingsMutation } =
  privacySettingsApi;

```

## File: connectfy-client/src/modules/settings/GeneralSettings/hooks/useGeneralSettings.tsx
```typescript
import { useAuthStore } from "@/store/zustand/useAuthStore";
import { useGetGeneralSettingsQuery } from "@/modules/settings/GeneralSettings/api/api.ts";
import { useGetMeQuery } from "@/modules/profile/api/api";

export function useGeneralSettings() {
  const { access_token } = useAuthStore();
  const { isMeLoaded } = useGetMeQuery(undefined, {
    skip: !access_token,
    selectFromResult: (result) => ({ isMeLoaded: result.isSuccess }),
  });

  const result = useGetGeneralSettingsQuery(undefined, {
    skip: !access_token || !isMeLoaded,
  });

  return {
    generalSettings: result.data,
    ...result,
  };
}

```

## File: connectfy-client/src/modules/settings/GeneralSettings/constants/contant.ts
```typescript
import {
  DATE_FORMAT,
  LANGUAGE,
  NOTIFICATION_CONTENT_MODE,
  NOTIFICATION_SOUND_MODE,
  PRIVACY_SETTINGS_CHOICE,
  STARTUP_PAGE,
  THEME,
  TIME_FORMAT,
} from "@/common/enums/enums";
import { TFunction } from "i18next";
import {
  Bell,
  Globe,
  MessageCircle,
  Radio,
  User,
  UserCircle,
  Users,
} from "lucide-react";
import { snack } from "@/common/utils/snackManager";
import { FormikProps } from "formik";
import { IEditGeneralSettings, IGeneralSettings } from "../types/types";

export const languageOptions = (
  t: TFunction,
  formik: FormikProps<IEditGeneralSettings>,
) => {
  const handleLanguageChange = (code: LANGUAGE) => {
    formik.setFieldValue("language", code);
  };

  return {
    title: t("common.language_selection"),
    activeKey: formik.values.language!,
    selections: [
      {
        key: "az",
        name: "Azərbaycan",
        icon: Globe,
        onClick: () => handleLanguageChange(LANGUAGE.AZ),
      },
      {
        key: "en",
        name: "English",
        icon: Globe,
        onClick: () => handleLanguageChange(LANGUAGE.EN),
      },
      {
        key: "ru",
        name: "Русский",
        icon: Globe,
        onClick: () => handleLanguageChange(LANGUAGE.RU),
      },
      {
        key: "tr",
        name: "Türkçe",
        icon: Globe,
        onClick: () => handleLanguageChange(LANGUAGE.TR),
      },
    ],
  };
};

export const homepageOptions = (
  t: TFunction,
  formik: FormikProps<IEditGeneralSettings>,
) => {
  return {
    title: t("common.homepage"),
    activeKey: formik.values.startupPage!,
    selections: [
      {
        key: STARTUP_PAGE.MESSENGER,
        name: t(`enum.${STARTUP_PAGE.MESSENGER}`),
        title: t(`enum.${STARTUP_PAGE.MESSENGER}`),
        icon: MessageCircle,
        onClick: () =>
          formik.setFieldValue("startupPage", STARTUP_PAGE.MESSENGER),
      },
      {
        key: STARTUP_PAGE.GROUPS,
        name: t(`enum.${STARTUP_PAGE.GROUPS}`),
        title: t(`enum.${STARTUP_PAGE.GROUPS}`),
        icon: Users,
        onClick: () => formik.setFieldValue("startupPage", STARTUP_PAGE.GROUPS),
      },
      {
        key: STARTUP_PAGE.CHANNELS,
        name: t(`enum.${STARTUP_PAGE.CHANNELS}`),
        title: t(`enum.${STARTUP_PAGE.CHANNELS}`),
        icon: Radio,
        onClick: () =>
          formik.setFieldValue("startupPage", STARTUP_PAGE.CHANNELS),
      },
      {
        key: STARTUP_PAGE.USERS,
        name: t(`enum.${STARTUP_PAGE.USERS}`),
        title: t(`enum.${STARTUP_PAGE.USERS}`),
        icon: UserCircle,
        onClick: () => formik.setFieldValue("startupPage", STARTUP_PAGE.USERS),
      },
      {
        key: STARTUP_PAGE.NOTIFICATIONS,
        name: t(`enum.${STARTUP_PAGE.NOTIFICATIONS}`),
        title: t(`enum.${STARTUP_PAGE.NOTIFICATIONS}`),
        icon: Bell,
        onClick: () =>
          formik.setFieldValue("startupPage", STARTUP_PAGE.NOTIFICATIONS),
      },
      {
        key: STARTUP_PAGE.PROFILE,
        name: t(`enum.${STARTUP_PAGE.PROFILE}`),
        title: t(`enum.${STARTUP_PAGE.PROFILE}`),
        icon: User,
        onClick: () =>
          formik.setFieldValue("startupPage", STARTUP_PAGE.PROFILE),
      },
    ],
  };
};

export const initialState = (data: IGeneralSettings): IEditGeneralSettings => {
  const { _id, theme, language, startupPage, timeZone } = data;

  const initialState = {
    _id,
    theme,
    language,
    startupPage,
    timeZone,
  };

  return initialState;
};

export const validateGenerateSettings = (
  values: IEditGeneralSettings,
  t: TFunction,
): void => {
  const { theme, language, startupPage, timeZone } = values;

  if (theme && !Object.values(THEME).includes(theme))
    snack.error(t("error_messages.invalid_choice"));

  if (language && !Object.values(LANGUAGE).includes(language))
    snack.error(t("error_messages.invalid_choice"));

  if (startupPage && !Object.values(STARTUP_PAGE).includes(startupPage))
    snack.error(t("error_messages.invalid_choice"));

  if (timeZone) {
    if (
      timeZone.dateFormat &&
      !Object.values(DATE_FORMAT).includes(timeZone.dateFormat)
    )
      snack.error(t("error_messages.invalid_choice"));

    if (
      timeZone.timeFormat &&
      !Object.values(TIME_FORMAT).includes(timeZone.timeFormat)
    )
      snack.error(t("error_messages.invalid_choice"));
  }
};

export const resetSettingsResponse = {
  generalSettings: {
    startupPage: STARTUP_PAGE.MESSENGER,
    timeZone: {
      timeFormat: TIME_FORMAT.H24,
      dateFormat: DATE_FORMAT.DDMMYYYY,
    },
  },
  privacySettings: {
    email: PRIVACY_SETTINGS_CHOICE.NOBODY,
    bio: PRIVACY_SETTINGS_CHOICE.EVERYONE,
    gender: PRIVACY_SETTINGS_CHOICE.NOBODY,
    location: PRIVACY_SETTINGS_CHOICE.EVERYONE,
    socialLinks: PRIVACY_SETTINGS_CHOICE.EVERYONE,
    lastSeen: PRIVACY_SETTINGS_CHOICE.EVERYONE,
    avatar: PRIVACY_SETTINGS_CHOICE.EVERYONE,
    messageRequest: PRIVACY_SETTINGS_CHOICE.EVERYONE,
    birthdayDate: PRIVACY_SETTINGS_CHOICE.EVERYONE,
    friendshipRequest: true,
    readReceipts: true,
  },
  notificationSettings: {
    notificationSoundMode: NOTIFICATION_SOUND_MODE.SOUND,
    notificationContentMode: NOTIFICATION_CONTENT_MODE.HEADER_AND_CONTENT,
    sendMessageSound: true,
    receiveMessageSound: true,
    privateMessageSound: true,
    groupMessageSound: true,
    systemNotificationSound: true,
    friendshipNotificationSound: true,
    showPrivateMessageNotification: true,
    showGroupMessageNotification: true,
    showFriendshipNotification: true,
    showSystemNotification: true,
  },
};

```

## File: connectfy-client/src/modules/settings/GeneralSettings/ui/GeneralSetttings.tsx
```typescript
import {
  Sun,
  Moon,
  Home,
  Clock,
  RefreshCw,
  MonitorSmartphone,
  SunMoon,
  Languages,
} from "lucide-react";
import "./generalSetttings.style.css";
import { Fragment, useEffect, useState } from "react";
import CustomSelect from "@/components/ui/Select/CustomSelect/CustomSelect";
import { useTheme } from "@/context/ThemeContext";
import { useTranslation } from "react-i18next";
import UniqueHeader from "@/components/Header/UnqiueHeader/UniqueHeader";
import SettingCard from "@/components/Card/SettingsCard/SettingCard";
import { ROUTER } from "@/common/constants/routet";
import {
  homepageOptions,
  initialState,
  languageOptions,
  validateGenerateSettings,
} from "../constants/contant";
import { useFormik } from "formik";
import { DATE_FORMAT, THEME, TIME_FORMAT } from "@/common/enums/enums";
import { snack } from "@/common/utils/snackManager";
import { useBlocker } from "@/hooks/useBlocker";
import { useBeforeUnload } from "@/hooks/useBeforeUnload";
import SaveChangesModal from "@/components/Modal/SaveChangesModal/SaveChangesModal";
import useBoolean from "@/hooks/useBoolean";
import ActionConfirmModal from "@/components/Modal/ActionConfirmModal/ActionConfirmModal";
import {
  useEditGeneralSettingsMutation,
  useResetSettingsMutation,
} from "../api/api";
import { useErrors } from "@/hooks/useErrors";
import { SettingsSkeleton } from "@/common/utils/skeleton";
import { getChangedData } from "@/common/utils/getDirtyValues";
import { IEditGeneralSettings } from "../types/types";
import Button from "@/components/ui/CustomButton/Button/Button";
import { useGeneralSettings } from "../hooks/useGeneralSettings";
import { useAppNavigation } from "@/hooks/useAppNavigation";

const GeneralSettings = () => {
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();
  const { theme, toggleTheme } = useTheme();
  const { showResponseErrors } = useErrors();
  const [selectedTheme, setSelectedTheme] = useState(theme);

  const { generalSettings, isLoading: LOADING_GET } = useGeneralSettings();
  const [updateGeneralSettings, { isLoading: LOADING_UPDATE }] =
    useEditGeneralSettingsMutation();
  const [resetSettings, { isLoading: LOADING_RESET_SETTINGS }] =
    useResetSettingsMutation();

  const resetSettingsModal = useBoolean();

  const formik = useFormik({
    initialValues: initialState(generalSettings!),
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    validate: (values) => validateGenerateSettings(values, t),
    onSubmit: async (values, { resetForm }) => {
      try {
        const changedData = getChangedData<IEditGeneralSettings>(
          generalSettings!,
          values,
        );
        values = {
          _id: generalSettings!._id,
          ...changedData,
        };
        const res = await updateGeneralSettings(values).unwrap();
        snack.success(
          t("user_messages.information_updated", { lng: res.language }),
        );
        resetForm();
      } catch (error) {
        showResponseErrors(error);
      }
    },
  });

  const { pending, confirm, cancel } = useBlocker(!!formik.dirty);
  useBeforeUnload(!!formik.dirty, t("common.unsaved_changes_message"));

  const isActiveTheme = (theme: THEME) => selectedTheme === theme;
  const LANGUAGE_OPTIONS = languageOptions(t, formik);
  const HOMEPAGE_OPTIONS = homepageOptions(t, formik);

  const changeAppTheme = (appTheme: THEME) => {
    if (theme === appTheme) return;
    formik.setFieldValue("theme", appTheme);
    toggleTheme(appTheme);
  };

  const getLabel = (options: any) =>
    options.selections.find((s: any) => s.key === options.activeKey)?.name ||
    options.title;

  const onClickBack = () => navigate(ROUTER.SETTINGS.MAIN);

  useEffect(() => {
    setSelectedTheme(theme);
  }, [theme]);

  const handleSaveAndLeave = async () => {
    await formik.submitForm();
    confirm();
  };

  const handleDiscardChanges = () => {
    if (formik.values.theme !== generalSettings?.theme) {
      formik.setFieldValue("theme", generalSettings?.theme);
      toggleTheme(generalSettings?.theme as THEME);
    }

    formik.resetForm();
    confirm();
  };

  const handleCancelModal = () => {
    cancel();
  };

  const handleResetSettings = async () => {
    try {
      await resetSettings().unwrap();
      snack.success(t("user_messages.information_updated"));

      if (formik.values.theme !== generalSettings?.theme) {
        formik.setFieldValue("theme", generalSettings?.theme);
        toggleTheme(generalSettings?.theme as THEME);
      }

      formik.resetForm();
      resetSettingsModal.onClose();
    } catch (error) {
      showResponseErrors(error);
    }
  };

  return (
    <Fragment>
      <section className="general-settings">
        <UniqueHeader
          headerTitle={t("common.general_settings")}
          headerSubtitle={t("common.configure_app_settings")}
          onClickBack={onClickBack}
          onHeaderButtonClick={formik.handleSubmit}
          showHeaderButton
          isHeaderButtonDisabled={!formik.dirty || LOADING_UPDATE}
          isLoading={LOADING_UPDATE}
        />
        <div className="general-settings-container">
          {LOADING_GET ? (
            <SettingsSkeleton count={5} />
          ) : (
            <Fragment>
              <div className="general-settings-content">
                <SettingCard
                  header={{
                    icon: SunMoon,
                    title: t("common.app_theme"),
                    subtitle: t("common.choose_light_dark_system"),
                  }}
                >
                  <div className="general-theme-options">
                    <div
                      className={`general-theme-option ${isActiveTheme(THEME.LIGHT) ? "active" : ""}`}
                      onClick={() => changeAppTheme(THEME.LIGHT)}
                    >
                      <div className="general-theme-option-icon">
                        <Sun size={24} />
                      </div>
                      <span>{t(`enum.${THEME.LIGHT}`)}</span>
                    </div>
                    <div
                      className={`general-theme-option ${isActiveTheme(THEME.DARK) ? "active" : ""}`}
                      onClick={() => changeAppTheme(THEME.DARK)}
                    >
                      <div className="general-theme-option-icon">
                        <Moon size={24} />
                      </div>
                      <span>{t(`enum.${THEME.DARK}`)}</span>
                    </div>
                    <div
                      className={`general-theme-option ${isActiveTheme(THEME.DEVICE) ? "active" : ""}`}
                      onClick={() => changeAppTheme(THEME.DEVICE)}
                    >
                      <div className="general-theme-option-icon">
                        <MonitorSmartphone size={24} />
                      </div>
                      <span>{t(`enum.${THEME.DEVICE}`)}</span>
                    </div>
                  </div>
                </SettingCard>

                <SettingCard
                  header={{
                    icon: Languages,
                    title: t("common.app_language"),
                    subtitle: t("common.select_app_language"),
                  }}
                >
                  <CustomSelect
                    buttonTitle={getLabel(LANGUAGE_OPTIONS)}
                    options={LANGUAGE_OPTIONS}
                  />
                </SettingCard>

                <SettingCard
                  header={{
                    icon: Home,
                    title: t("common.startup_page"),
                    subtitle: t("common.which_page_on_start"),
                  }}
                >
                  <CustomSelect
                    buttonTitle={getLabel(HOMEPAGE_OPTIONS)}
                    options={HOMEPAGE_OPTIONS}
                  />
                </SettingCard>

                <SettingCard
                  header={{
                    icon: Clock,
                    title: t("common.timezone_and_format"),
                    subtitle: t("common.select_time_display_format"),
                  }}
                >
                  <Fragment>
                    <div style={{ marginBottom: "12px" }}>
                      <p
                        style={{
                          fontSize: "13px",
                          fontWeight: "600",
                          color: "#64748b",
                          margin: "0 0 8px 0",
                        }}
                      >
                        {t("common.time_format")}
                      </p>
                      <div className="general-time-format-options">
                        <div
                          className={`general-format-option ${
                            formik.values.timeZone?.timeFormat ===
                            TIME_FORMAT.H24
                              ? "active"
                              : ""
                          }`}
                          onClick={() =>
                            formik.setFieldValue(
                              "timeZone.timeFormat",
                              TIME_FORMAT.H24,
                            )
                          }
                        >
                          {t("common.format_24_hour_example")}
                        </div>
                        <div
                          className={`general-format-option ${
                            formik.values.timeZone?.timeFormat ===
                            TIME_FORMAT.H12
                              ? "active"
                              : ""
                          }`}
                          onClick={() =>
                            formik.setFieldValue(
                              "timeZone.timeFormat",
                              TIME_FORMAT.H12,
                            )
                          }
                        >
                          {t("common.format_12_hour_example")}
                        </div>
                      </div>
                    </div>

                    <div>
                      <p
                        style={{
                          fontSize: "13px",
                          fontWeight: "600",
                          color: "#64748b",
                          margin: "0 0 8px 0",
                        }}
                      >
                        {t("common.date_format")}
                      </p>
                      <div className="general-time-format-options">
                        <div
                          className={`general-format-option ${
                            formik.values.timeZone?.dateFormat ===
                            DATE_FORMAT.DDMMYYYY
                              ? "active"
                              : ""
                          }`}
                          onClick={() =>
                            formik.setFieldValue(
                              "timeZone.dateFormat",
                              DATE_FORMAT.DDMMYYYY,
                            )
                          }
                        >
                          {t("common.date_dd_mm_yyyy")}
                        </div>
                        <div
                          className={`general-format-option ${
                            formik.values.timeZone?.dateFormat ===
                            DATE_FORMAT.MMDDYYYY
                              ? "active"
                              : ""
                          }`}
                          onClick={() =>
                            formik.setFieldValue(
                              "timeZone.dateFormat",
                              DATE_FORMAT.MMDDYYYY,
                            )
                          }
                        >
                          {t("common.date_mm_dd_yyyy")}
                        </div>
                      </div>
                    </div>
                  </Fragment>
                </SettingCard>

                <SettingCard
                  header={{
                    icon: RefreshCw,
                    title: t("common.reset_to_defaults"),
                    subtitle: t("common.reset_all_general_settings"),
                  }}
                >
                  <Button
                    className="general-reset-button"
                    onClick={resetSettingsModal.onOpen}
                  >
                    <RefreshCw size={18} />
                    {t("common.reset_settings")}
                  </Button>
                </SettingCard>
              </div>
            </Fragment>
          )}
        </div>
      </section>

      <SaveChangesModal
        open={!!pending}
        handleSave={handleSaveAndLeave}
        handleCancel={handleCancelModal}
        handleDiscardChanges={handleDiscardChanges}
        isLoading={LOADING_UPDATE}
      />

      <ActionConfirmModal
        open={resetSettingsModal.open}
        onClose={resetSettingsModal.onClose}
        onCancel={resetSettingsModal.onClose}
        onConfirm={handleResetSettings}
        header={{ title: t("common.reset_settings_title") }}
        cancelBtn={{ title: t("common.cancel") }}
        confirmBtn={{ title: t("common.reset"), color: "var(--error-color)" }}
        isLoading={LOADING_RESET_SETTINGS}
      >
        <div className="mt-4 space-y-4">
          <p className="text-center text-[15px] text-(--muted-color)">
            {t("common.reset_settings_subtitle")}
          </p>

          <div className="grid grid-cols-1 gap-2">
            {[
              t("common.general_settings_reset"),
              t("common.privacy_settings"),
              t("common.notification_settings"),
            ].map((item, index) => (
              <span
                key={item}
                className="text-sm mb-1 font-medium text-(--text-color) italic"
              >
                {index + 1}. {item}
              </span>
            ))}
          </div>
        </div>
      </ActionConfirmModal>
    </Fragment>
  );
};

export default GeneralSettings;

```

## File: connectfy-client/src/modules/settings/GeneralSettings/router/router.tsx
```typescript
import { RouteObject } from "react-router-dom";
import { lazy } from "react";
import { ROUTER } from "@/common/constants/routet";
import ComponentLoader from "@/components/Loader/Components/ComponentLoader";

const GeneralSettings = ComponentLoader(
  lazy(() => import("../ui/GeneralSetttings")),
);

const routes: RouteObject[] = [
  {
    path: ROUTER.SETTINGS.GENERAL,
    element: <GeneralSettings />,
  },
];

export default routes;

```

## File: connectfy-client/src/modules/settings/GeneralSettings/types/types.ts
```typescript
import {
  DATE_FORMAT,
  LANGUAGE,
  STARTUP_PAGE,
  THEME,
  TIME_FORMAT,
} from "@/common/enums/enums";
import { IPrivacySettings } from "../../PrivacySettings/types/types";
import { INotificationSettings } from "../../NotificationSettings/types/types";

export interface ITimeZone {
  timeFormat: TIME_FORMAT;
  dateFormat: DATE_FORMAT;
}

export interface IGeneralSettings {
  _id: string;
  userId: string;
  theme: THEME;
  language: LANGUAGE;
  startupPage: STARTUP_PAGE;
  timeZone: ITimeZone;
}

export interface IEditGeneralSettings
  extends Partial<Omit<IGeneralSettings, "userId">> {
  _id: string;
}

export interface IResetSettings {
  generalSettings: IGeneralSettings;
  privacySettings: IPrivacySettings;
  notificationSettings: INotificationSettings;
}

```

## File: connectfy-client/src/modules/settings/GeneralSettings/api/api.ts
```typescript
import { baseQuery } from "@/common/api/axiosBaseQuery";
import { RESOURCE } from "@/common/enums/enums";
import { createApi } from "@reduxjs/toolkit/query/react";
import { IEditGeneralSettings, IGeneralSettings } from "../types/types";
import { API_ENDPOINTS } from "@/common/constants/apiEndpoints";
import { notificationSettingsApi } from "../../NotificationSettings/api/api";
import { privacySettingsApi } from "../../PrivacySettings/api/api";
import { IUpdateResponse } from "@/common/interfaces/interfaces";
import { resetSettingsResponse } from "../constants/contant";

export const generalSettingsApi = createApi({
  reducerPath: RESOURCE.GENERAL_SETTINGS,
  baseQuery: baseQuery,
  tagTypes: ["GeneralSettings", "NotificationSettings", "PrivacySettings"],
  endpoints: (builder) => ({
    // ====================== GET GENERAL SETTINGS
    getGeneralSettings: builder.query<IGeneralSettings, void>({
      query: () => ({
        url: API_ENDPOINTS.ACCOUNT.SETTINGS.GENERAL_SETTINGS.GET,
        method: "POST",
      }),
      providesTags: ["GeneralSettings"],
    }),

    // ====================== EDIT GENERAL SETTINGS
    editGeneralSettings: builder.mutation<
      IUpdateResponse,
      IEditGeneralSettings
    >({
      query: (data) => ({
        url: API_ENDPOINTS.ACCOUNT.SETTINGS.GENERAL_SETTINGS.UPDATE,
        method: "PATCH",
        body: data,
      }),
      onQueryStarted: async (arg, { dispatch, queryFulfilled }) => {
        const { _id, ...updatedFields } = arg;

        const patchResult = dispatch(
          generalSettingsApi.util.updateQueryData(
            "getGeneralSettings",
            undefined,
            (draft) => {
              if (draft) {
                Object.assign(draft, updatedFields);
              }
            },
          ),
        );

        try {
          await queryFulfilled;
        } catch (error) {
          patchResult.undo();
        }
      },
    }),

    // ====================== RESET SETTINGS
    resetSettings: builder.mutation<IUpdateResponse, void>({
      query: () => ({
        url: API_ENDPOINTS.ACCOUNT.SETTINGS.GENERAL_SETTINGS.RESET,
        method: "PATCH",
      }),

      async onQueryStarted(_, { dispatch, queryFulfilled }) {
        const generalPatchResult = dispatch(
          generalSettingsApi.util.updateQueryData(
            "getGeneralSettings",
            undefined,
            (draft) => {
              if (draft) {
                Object.assign(draft, resetSettingsResponse.generalSettings);
              }
            },
          ),
        );

        const notificationPatchResult = dispatch(
          notificationSettingsApi.util.updateQueryData(
            "getNotificationSettings",
            undefined,
            (draft) => {
              if (draft) {
                Object.assign(
                  draft,
                  resetSettingsResponse.notificationSettings,
                );
              }
            },
          ),
        );

        const privacyPatchResult = dispatch(
          privacySettingsApi.util.updateQueryData(
            "getPrivacySettings",
            undefined,
            (draft) => {
              if (draft) {
                Object.assign(draft, resetSettingsResponse.privacySettings);
              }
            },
          ),
        );

        try {
          await queryFulfilled;
        } catch (error) {
          generalPatchResult.undo();
          notificationPatchResult.undo();
          privacyPatchResult.undo();
        }
      },
      invalidatesTags: [
        "GeneralSettings",
        "NotificationSettings",
        "PrivacySettings",
      ],
    }),
  }),
});

export const {
  useGetGeneralSettingsQuery,
  useEditGeneralSettingsMutation,
  useResetSettingsMutation,
} = generalSettingsApi;

```

## File: connectfy-client/src/modules/settings/Settings/ui/Settings.tsx
```typescript
import { useTranslation } from "react-i18next";
import {
  Settings as SettingsIcon,
  KeyRound,
  Paintbrush,
  UserCog,
  Bell,
  Keyboard,
} from "lucide-react";
import { ROUTER } from "@/common/constants/routet";
import { memo } from "react";
import { useAppNavigation } from "@/hooks/useAppNavigation";

const Settings = () => {
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();

  const settingsOptions = [
    {
      title: t("common.general"),
      description: t("common.general_description"),
      icon: <SettingsIcon size={24} />,
      gradient: "from-blue-500 to-blue-600",
      path: ROUTER.SETTINGS.GENERAL,
    },
    {
      title: t("common.account"),
      description: t("common.account_description"),
      icon: <UserCog size={24} />,
      gradient: "from-pink-500 to-pink-600",
      path: ROUTER.SETTINGS.ACCOUNT,
    },
    {
      title: t("common.privacy"),
      description: t("common.privacy_description"),
      icon: <KeyRound size={24} />,
      gradient: "from-violet-500 to-purple-600",
      path: ROUTER.SETTINGS.PRIVACY,
    },
    {
      title: t("common.notifications"),
      description: t("common.notification_description"),
      icon: <Bell size={24} />,
      gradient: "from-sky-400 to-blue-600",
      path: ROUTER.SETTINGS.NOTIFICATION,
    },
    {
      title: t("common.appearance"),
      description: t("common.appearance_description"),
      icon: <Paintbrush size={24} />,
      gradient: "from-amber-400 to-orange-600",
      path: ROUTER.SETTINGS.BACKGROUND,
    },
    {
      title: t("common.keyboard_shortcuts"),
      description: t("common.keyboard_shortcuts_description"),
      icon: <Keyboard size={24} />,
      gradient: "from-indigo-500 to-indigo-700",
      path: ROUTER.SETTINGS.SHORTCUT,
    },
  ];

  return (
    <div className="w-full min-h-screen p-6 transition-colors duration-300 bg-(--bg-color)">
      <div className="max-w-[900px] mx-auto">
        {/* Header / Hero Section */}
        <div className="text-center mb-12 animate-in fade-in slide-in-from-top-4 duration-700">
          <div className="w-24 h-24 mx-auto mb-6 flex items-center justify-center text-white rounded-[24px] bg-(--primary-color) shadow-(--active-shadow)">
            <SettingsIcon size={48} strokeWidth={2} />
          </div>
          <h1 className="text-4xl font-extrabold mb-3 text-(--text-primary)">
            {t("common.settings_title")}
          </h1>
          <p className="max-w-[600px] mx-auto text-lg leading-relaxed text-(--muted-color)">
            {t("common.settings_description")}
          </p>
        </div>

        {/* Settings Grid */}
        <div className="grid grid-cols-1 md:grid-cols-2 gap-5 mb-8">
          {settingsOptions.map((option, index) => (
            <div
              key={index}
              role="button"
              tabIndex={0}
              onClick={() => navigate(option.path)}
              className="group flex items-center gap-4 p-6 rounded-2xl cursor-pointer transition-all duration-300 border border-transparent 
                         bg-(--card-bg) shadow-(--card-shadow) 
                         hover:shadow-xl hover:-translate-y-1 hover:border-(--primary-color)
                         active:scale-[0.98]"
            >
              {/* Icon Container */}
              <div
                className={`w-14 h-14 shrink-0 rounded-xl flex items-center justify-center text-white bg-linear-to-br ${option.gradient} shadow-inner`}
              >
                {option.icon}
              </div>

              {/* Text Info */}
              <div className="text-left">
                <h3 className="text-lg font-bold mb-1 transition-colors text-(--text-primary) group-hover:text-(--primary-color)">
                  {option.title}
                </h3>
                <p className="text-sm leading-snug text-(--muted-color)">
                  {option.description}
                </p>
              </div>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};

export default memo(Settings);

```

## File: connectfy-client/src/modules/settings/Settings/router/router.tsx
```typescript
import { RouteObject } from "react-router-dom";
import { lazy } from "react";
import { ROUTER } from "@/common/constants/routet";
import ComponentLoader from "@/components/Loader/Components/ComponentLoader";

const Settings = ComponentLoader(lazy(() => import("../ui/Settings")));

const routes: RouteObject[] = [
  {
    path: ROUTER.SETTINGS.MAIN,
    element: <Settings />,
  },
];

export default routes;

```

## File: connectfy-client/src/modules/settings/NotificationSettings/hooks/useNotificationSettings.tsx
```typescript
import { useAuthStore } from "@/store/zustand/useAuthStore";
import { useGetNotificationSettingsQuery } from "@/modules/settings/NotificationSettings/api/api.ts";
import { useGetMeQuery } from "@/modules/profile/api/api";

export function useNotificationSettings() {
  const { access_token } = useAuthStore();
  const { isMeLoaded } = useGetMeQuery(undefined, {
    skip: !access_token,
    selectFromResult: (result) => ({ isMeLoaded: result.isSuccess }),
  });

  const result = useGetNotificationSettingsQuery(undefined, {
    skip: !access_token || !isMeLoaded,
  });

  return {
    notificationSettings: result.data,
    ...result,
  };
}

```

## File: connectfy-client/src/modules/settings/NotificationSettings/constants/constant.ts
```typescript
import {
  NOTIFICATION_CONTENT_MODE,
  NOTIFICATION_SOUND_MODE,
} from "@/common/enums/enums";
import { snack } from "@/common/utils/snackManager";
import { TFunction } from "i18next";
import {
  Bell,
  Cog,
  MessageSquare,
  UserPlus,
  Users,
  Volume2,
} from "lucide-react";
import {
  IEditNotificationSettings,
  INotificationSettings,
} from "../types/types";

export const notificationModeOptions = (t: TFunction) => {
  return [
    {
      key: NOTIFICATION_SOUND_MODE.SOUND,
      name: t("common.sound_notifications"),
      description: t("common.all_notifications_with_sound"),
    },
    {
      key: NOTIFICATION_SOUND_MODE.SILENT,
      name: t("common.silent_mode"),
      description: t("common.only_visual_no_sound"),
    },
    {
      key: NOTIFICATION_SOUND_MODE.DND,
      name: t("common.do_not_disturb"),
      description: t("common.no_notifications"),
    },
  ];
};

export const notificationContentOptions = (t: TFunction) => {
  return [
    {
      key: NOTIFICATION_CONTENT_MODE.HEADER_AND_CONTENT,
      name: t("common.show_header_and_content"),
      description: t("common.show_header_and_content_desc"),
    },
    {
      key: NOTIFICATION_CONTENT_MODE.HEADER_ONLY,
      name: t("common.show_header_only"),
      description: t("common.show_header_only_desc"),
    },
    {
      key: NOTIFICATION_CONTENT_MODE.HIDE_NOTIFICATION,
      name: t("common.hide_notifications"),
      description: t("common.hide_notifications_desc"),
    },
  ];
};

export const NOTIFICATION_FIELDS = (t: TFunction) => ({
  messageSounds: [
    {
      field: "sendMessageSound",
      title: t("common.sent_sound"),
      desc: t("common.sent_sound_desc"),
      icon: MessageSquare,
    },
    {
      field: "receiveMessageSound",
      title: t("common.receive_sound"),
      desc: t("common.receive_sound_desc"),
      icon: Volume2,
    },
  ],
  notificationSounds: [
    {
      field: "privateMessageSound",
      title: t("common.private_message_sound"),
      desc: t("common.private_message_sound_desc"),
      icon: MessageSquare,
    },
    {
      field: "groupMessageSound",
      title: t("common.group_message_sound"),
      desc: t("common.group_message_sound_desc"),
      icon: Users,
    },
    {
      field: "systemNotificationSound",
      title: t("common.system_notification_sound"),
      desc: t("common.system_notification_sound_desc"),
      icon: Cog,
    },
    {
      field: "friendshipNotificationSound",
      title: t("common.friendship_notification_sound"),
      desc: t("common.friendship_notification_sound_desc"),
      icon: UserPlus,
    },
  ],
  notificationBanners: [
    {
      field: "showPrivateMessageNotification",
      title: t("common.show_private_message_notification"),
      desc: t("common.show_private_message_notification_desc"),
      icon: MessageSquare,
    },
    {
      field: "showGroupMessageNotification",
      title: t("common.show_group_message_notification"),
      desc: t("common.show_group_message_notification_desc"),
      icon: Users,
    },
    {
      field: "showFriendshipNotification",
      title: t("common.show_friendship_notification"),
      desc: t("common.show_friendship_notification_desc"),
      icon: UserPlus,
    },
    {
      field: "showSystemNotification",
      title: t("common.show_system_notification"),
      desc: t("common.show_system_notification_desc"),
      icon: Cog,
    },
  ],
});

export const initialState = (
  data: INotificationSettings,
): IEditNotificationSettings => {
  const {
    _id,
    notificationSoundMode,
    notificationContentMode,
    sendMessageSound,
    receiveMessageSound,
    privateMessageSound,
    groupMessageSound,
    systemNotificationSound,
    friendshipNotificationSound,
    showPrivateMessageNotification,
    showGroupMessageNotification,
    showFriendshipNotification,
    showSystemNotification,
  } = data;

  return {
    _id,
    notificationSoundMode,
    notificationContentMode,
    sendMessageSound,
    receiveMessageSound,
    privateMessageSound,
    groupMessageSound,
    systemNotificationSound,
    friendshipNotificationSound,
    showPrivateMessageNotification,
    showGroupMessageNotification,
    showFriendshipNotification,
    showSystemNotification,
  };
};

export const validateNotificationSettings = (
  values: IEditNotificationSettings,
  t: TFunction,
): void => {
  const { notificationSoundMode, notificationContentMode } = values;

  if (
    notificationSoundMode &&
    !Object.values(NOTIFICATION_SOUND_MODE).includes(notificationSoundMode)
  ) {
    snack.error(t("error_messages.invalid_choice"));
    return;
  }

  if (
    notificationContentMode &&
    !Object.values(NOTIFICATION_CONTENT_MODE).includes(notificationContentMode)
  ) {
    snack.error(t("error_messages.invalid_choice"));
    return;
  }
};

```

## File: connectfy-client/src/modules/settings/NotificationSettings/ui/NotificationSettings.tsx
```typescript
import { Bell } from "lucide-react";
import { Fragment } from "react";
import "./notificationSettings.style.css";
import { useTranslation } from "react-i18next";
import UniqueHeader from "@/components/Header/UnqiueHeader/UniqueHeader";
import ToggleCard from "@/components/Card/ToggleCard/ToggleCard";
import { ROUTER } from "@/common/constants/routet";
import SettingCard from "@/components/Card/SettingsCard/SettingCard";
import {
  initialState,
  NOTIFICATION_FIELDS,
  notificationModeOptions,
  notificationContentOptions,
  validateNotificationSettings,
} from "../constants/constant";
import { useFormik } from "formik";
import { snack } from "@/common/utils/snackManager";
import { useBlocker } from "@/hooks/useBlocker";
import { useBeforeUnload } from "@/hooks/useBeforeUnload";
import SaveChangesModal from "@/components/Modal/SaveChangesModal/SaveChangesModal";
import { useEditNotificationSettingsMutation } from "../api/api";
import { useErrors } from "@/hooks/useErrors";
import { SettingsSkeleton } from "@/common/utils/skeleton";
import { getChangedData } from "@/common/utils/getDirtyValues";
import { IEditNotificationSettings } from "../types/types";
import { useNotificationSettings } from "../hooks/useNotificationSettings";
import RadioGroup from "@/components/ui/CustomRadio/RadioGroup";
import {
  NOTIFICATION_CONTENT_MODE,
  NOTIFICATION_SOUND_MODE,
} from "@/common/enums/enums";
import { useAppNavigation } from "@/hooks/useAppNavigation";

const NotificationSettings = () => {
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();

  const { notificationSettings, isLoading: LOADING_GET } =
    useNotificationSettings();
  const [editNotificationSettings, { isLoading: LOADING_UPDATE }] =
    useEditNotificationSettingsMutation();

  const { showResponseErrors } = useErrors();

  const formik = useFormik({
    initialValues: initialState(notificationSettings!),
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    validate: (values) => validateNotificationSettings(values, t),
    onSubmit: async (values, { resetForm }) => {
      try {
        const changedData = getChangedData<IEditNotificationSettings>(
          notificationSettings!,
          values,
        );
        values = {
          _id: notificationSettings!._id,
          ...changedData,
        };
        await editNotificationSettings(values).unwrap();
        snack.success(t("user_messages.information_updated"));
        resetForm();
      } catch (error) {
        showResponseErrors(error);
      }
    },
  });

  const { pending, confirm, cancel } = useBlocker(!!formik.dirty);
  useBeforeUnload(!!formik.dirty, t("common.unsaved_changes_message"));

  const handleSaveAndLeave = async () => {
    await formik.submitForm();
    confirm();
  };

  const handleDiscardChanges = () => {
    formik.resetForm();
    confirm();
  };

  const handleCancelModal = () => {
    cancel();
  };

  const NOTIFICATION_MODE_OPTIONS = notificationModeOptions(t);
  const NOTIFICATION_CONTENT_OPTIONS = notificationContentOptions(t);
  const notificationFields = NOTIFICATION_FIELDS(t);

  const onClickBack = () => navigate(ROUTER.SETTINGS.MAIN);

  return (
    <Fragment>
      <section className="notification-settings">
        <UniqueHeader
          headerTitle={t("common.notification_header")}
          headerSubtitle={t("common.notification_subheader")}
          onClickBack={onClickBack}
          onHeaderButtonClick={formik.handleSubmit}
          showHeaderButton
          isHeaderButtonDisabled={
            !formik.dirty || LOADING_UPDATE || LOADING_GET
          }
          isLoading={LOADING_UPDATE}
        />
        <div className="notification-settings-container">
          {LOADING_GET ? (
            <SettingsSkeleton count={8} />
          ) : (
            <Fragment>
              <div className="notification-settings-content">
                <div className="notification-privacy-section">
                  <h2 className="notification-section-title">
                    {t("common.general_notification_title")}
                  </h2>

                  <SettingCard
                    header={{
                      icon: Bell,
                      title: t("common.notification_mode"),
                      subtitle: t("common.select_default_notification"),
                    }}
                  >
                    <div className="general-notification-modes">
                      <RadioGroup
                        name="notificationSoundMode"
                        options={NOTIFICATION_MODE_OPTIONS}
                        value={
                          formik.values
                            .notificationSoundMode as NOTIFICATION_SOUND_MODE
                        }
                        onChange={formik.setFieldValue}
                      />
                    </div>
                  </SettingCard>

                  <SettingCard
                    header={{
                      icon: Bell,
                      title: t("common.notification_content"),
                      subtitle: t("common.select_default_content"),
                    }}
                  >
                    <div className="general-notification-modes">
                      <RadioGroup
                        name="notificationContentMode"
                        options={NOTIFICATION_CONTENT_OPTIONS}
                        value={
                          formik.values
                            .notificationContentMode as NOTIFICATION_CONTENT_MODE
                        }
                        onChange={formik.setFieldValue}
                      />
                    </div>
                  </SettingCard>
                </div>

                {/* MESSAGE SOUND SETTINGS */}
                <div className="notification-privacy-section">
                  <h2 className="notification-section-title">
                    {t("common.message_sound_section_title")}
                  </h2>

                  {notificationFields.messageSounds.map((field) => {
                    const Icon = field.icon;
                    return (
                      <ToggleCard
                        key={field.field}
                        header={{
                          icon: Icon,
                          title: field.title,
                          subtitle: field.desc,
                        }}
                        slider={{
                          checked:
                            !!formik.values[
                              field.field as keyof typeof formik.values
                            ],
                          onClick: () =>
                            formik.setFieldValue(
                              field.field,
                              !formik.values[
                                field.field as keyof typeof formik.values
                              ],
                            ),
                        }}
                      />
                    );
                  })}
                </div>

                {/* NOTIFICATION SOUND SETTINGS */}
                <div className="notification-privacy-section">
                  <h2 className="notification-section-title">
                    {t("common.sound_section_title")}
                  </h2>

                  {notificationFields.notificationSounds.map((field) => {
                    const Icon = field.icon;
                    return (
                      <ToggleCard
                        key={field.field}
                        header={{
                          icon: Icon,
                          title: field.title,
                          subtitle: field.desc,
                        }}
                        slider={{
                          checked:
                            !!formik.values[
                              field.field as keyof typeof formik.values
                            ],
                          onClick: () =>
                            formik.setFieldValue(
                              field.field,
                              !formik.values[
                                field.field as keyof typeof formik.values
                              ],
                            ),
                        }}
                      />
                    );
                  })}
                </div>

                {/* NOTIFICATION BANNER SETTINGS */}
                <div className="notification-privacy-section">
                  <h2 className="notification-section-title">
                    {t("common.banner_section_title")}
                  </h2>

                  {notificationFields.notificationBanners.map((field) => {
                    const Icon = field.icon;
                    return (
                      <ToggleCard
                        key={field.field}
                        header={{
                          icon: Icon,
                          title: field.title,
                          subtitle: field.desc,
                        }}
                        slider={{
                          checked:
                            !!formik.values[
                              field.field as keyof typeof formik.values
                            ],
                          onClick: () =>
                            formik.setFieldValue(
                              field.field,
                              !formik.values[
                                field.field as keyof typeof formik.values
                              ],
                            ),
                        }}
                      />
                    );
                  })}
                </div>
              </div>
            </Fragment>
          )}
        </div>

        <SaveChangesModal
          open={!!pending}
          handleSave={handleSaveAndLeave}
          handleCancel={handleCancelModal}
          handleDiscardChanges={handleDiscardChanges}
          isLoading={LOADING_UPDATE}
        />
      </section>
    </Fragment>
  );
};

export default NotificationSettings;

```

## File: connectfy-client/src/modules/settings/NotificationSettings/router/router.tsx
```typescript
import { RouteObject } from "react-router-dom";
import { lazy } from "react";
import { ROUTER } from "@/common/constants/routet";
import ComponentLoader from "@/components/Loader/Components/ComponentLoader";

const NotificationSettings = ComponentLoader(
  lazy(() => import("../ui/NotificationSettings")),
);

const routes: RouteObject[] = [
  {
    path: ROUTER.SETTINGS.NOTIFICATION,
    element: <NotificationSettings />,
  },
];

export default routes;

```

## File: connectfy-client/src/modules/settings/NotificationSettings/types/types.ts
```typescript
import {
  NOTIFICATION_CONTENT_MODE,
  NOTIFICATION_SOUND_MODE,
} from "@/common/enums/enums";

export interface INotificationSettings {
  _id: string;
  userId: string;
  notificationSoundMode: NOTIFICATION_SOUND_MODE;
  notificationContentMode: NOTIFICATION_CONTENT_MODE;
  sendMessageSound: boolean;
  receiveMessageSound: boolean;
  privateMessageSound: boolean;
  groupMessageSound: boolean;
  systemNotificationSound: boolean;
  friendshipNotificationSound: boolean;
  showPrivateMessageNotification: boolean;
  showGroupMessageNotification: boolean;
  showFriendshipNotification: boolean;
  showSystemNotification: boolean;
}

export interface IEditNotificationSettings
  extends Partial<Omit<INotificationSettings, "userId">> {
  _id: string;
}
```

## File: connectfy-client/src/modules/settings/NotificationSettings/api/api.ts
```typescript
import { baseQuery } from "@/common/api/axiosBaseQuery";
import { RESOURCE } from "@/common/enums/enums";
import { createApi } from "@reduxjs/toolkit/query/react";
import {
  IEditNotificationSettings,
  INotificationSettings,
} from "../types/types";
import { API_ENDPOINTS } from "@/common/constants/apiEndpoints";
import { IUpdateResponse } from "@/common/interfaces/interfaces";

export const notificationSettingsApi = createApi({
  reducerPath: RESOURCE.NOTIFICATION_SETTINGS,
  baseQuery: baseQuery,
  tagTypes: ["NotificationSettings"],
  endpoints: (builder) => ({
    // ====================== GET NOTIFICATION SETTINGS
    getNotificationSettings: builder.query<INotificationSettings, void>({
      query: () => ({
        url: API_ENDPOINTS.ACCOUNT.SETTINGS.NOTIFICATION_SETTINGS.GET,
        method: "POST",
      }),
      providesTags: ["NotificationSettings"],
    }),

    // ====================== EDIT NOTIFICATION SETTINGS
    editNotificationSettings: builder.mutation<
      IUpdateResponse,
      IEditNotificationSettings
    >({
      query: (data) => ({
        url: API_ENDPOINTS.ACCOUNT.SETTINGS.NOTIFICATION_SETTINGS.UPDATE,
        method: "PATCH",
        body: data,
      }),
      onQueryStarted: async (arg, { dispatch, queryFulfilled }) => {
        const { _id, ...updatedFields } = arg;

        const patchResult = dispatch(
          notificationSettingsApi.util.updateQueryData(
            "getNotificationSettings",
            undefined,
            (draft) => {
              if (draft) {
                Object.assign(draft, updatedFields);
              }
            },
          ),
        );

        try {
          await queryFulfilled;
        } catch (error) {
          patchResult.undo();
        }
      },
    }),
  }),
});

export const {
  useGetNotificationSettingsQuery,
  useEditNotificationSettingsMutation,
} = notificationSettingsApi;

```

## File: connectfy-client/src/modules/settings/AccountSettings/ui/AccountSettings.tsx
```typescript
import {
  FC,
  Fragment,
  useCallback,
  useEffect,
  useMemo,
  useRef,
  useState,
} from "react";
import {
  User,
  Mail,
  Key,
  Phone,
  Calendar,
  Trash2,
  Clock,
  AlertTriangle,
  LogOutIcon,
  TriangleAlert,
  ShieldCheck,
  ChevronRight,
} from "lucide-react";
import "./accountSettings.style.css";
import { useTranslation } from "react-i18next";
import UniqueHeader from "@/components/Header/UnqiueHeader/UniqueHeader";
import SettingCard from "@/components/Card/SettingsCard/SettingCard";
import { useSearchParams } from "react-router-dom";
import { ROUTER } from "@/common/constants/routet";
import {
  DATE_FORMAT,
  LOCAL_STORAGE_KEYS,
  PROVIDER,
  TIME_FORMAT,
  TOKEN_TYPE,
  TWO_FACTOR_ACTION,
} from "@/common/enums/enums";
import AuthenticateModal from "@/components/Modal/AuthenticateModal/AuthenticateModal";
import UsernameModal from "./components/Modal/UsernameModal/UsernameModal";
import ChangeEmailModal from "./components/Modal/EmailModal/ChangeEmailModal/ChangeEmailModal";
import PasswordModal from "./components/Modal/PasswordModal/PasswordModal";
import { ChangeModalKey } from "../types/types";
import { jwtDecode } from "jwt-decode";
import { snack } from "@/common/utils/snackManager";
import useBoolean from "@/hooks/useBoolean";
import VerifyChangeEmailModal from "./components/Modal/EmailModal/VerifyChangeEmailModal/VerifyChangeEmailModal";
import PhoneNumberModal from "./components/Modal/PhoneNumberModal/PhoneNumberModal";
import {
  DDMMMMYYYY,
  formatPhoneNumber,
  showDateWithHour,
} from "@/common/utils/formatValues";
import ActionConfirmModal from "@/components/Modal/ActionConfirmModal/ActionConfirmModal";
import DeleteAccountModal from "./components/Modal/DeleteAccountModal/DeleteAccountModal";
import { useAuthStore } from "@/store/zustand/useAuthStore";
import { useUser } from "@/context/UserContext";
import { useProfile } from "@/modules/profile/hooks/useProfile";
import {
  useDeactivateAccountMutation,
  useLogoutMutation,
  useVerifyChangeEmailMutation,
} from "../api/api";
import { useErrors } from "@/hooks/useErrors";
import { useTheme } from "@/context/ThemeContext";
import { SettingsSkeleton } from "@/common/utils/skeleton";
import { useDispatch } from "react-redux";
import { COUNTRIES } from "@/common/constants/constants";
import Button from "@/components/ui/CustomButton/Button/Button";
import TwoFactorModal from "./components/Modal/TwoFactorModal/TwoFactorModal";
import { useGeneralSettings } from "../../GeneralSettings/hooks/useGeneralSettings";
import { useAppNavigation } from "@/hooks/useAppNavigation";

const AccountSettings: FC = () => {
  const dispatch = useDispatch();
  const { t, i18n } = useTranslation();
  const { navigate } = useAppNavigation();
  const { authenticateToken, clear } = useAuthStore();
  const [searchParams, setSearchParams] = useSearchParams();

  const processedTokenRef = useRef<string | null>(null);
  const { showResponseErrors } = useErrors();
  const { theme } = useTheme();

  const { user, isLoading: LOADING_USER } = useUser();

  const { provider, isTwoFactorEnabled } = user ?? {};
  const usesOAuth =
    provider === PROVIDER.GOOGLE || provider === PROVIDER.FACEBOOK;
  const hasPhoneNumber = !!user?.phoneNumber?.number;

  const { profile, isLoading: LOADING_ACCOUNT } = useProfile();
  const { generalSettings, isLoading: LOADING_GENERAL_SETTINGS } =
    useGeneralSettings();

  const [logout, { isLoading: LOADING_LOGOUT }] = useLogoutMutation();
  const [deactivateAccount, { isLoading: LOADING_DEACTIVATE_ACCOUNT }] =
    useDeactivateAccountMutation();
  const [
    verifyChangeEmail,
    {
      isLoading: LOADING_VERIFY_CHANGE_EMAIL,
      isError: isVerifyChangeEmailFailed,
    },
  ] = useVerifyChangeEmailMutation();

  const [authModal, setAuthModal] = useState<{
    open: boolean;
    authType?: TOKEN_TYPE;
    next?: ChangeModalKey;
  }>({ open: false, authType: undefined, next: null });

  const [openModal, setOpenModal] = useState<ChangeModalKey>(null);

  const {
    open: verifiedModalOpen,
    onClose: onVerifiedModalClose,
    onOpen: onVerifiedOpen,
  } = useBoolean();

  const {
    open: logoutModalOpen,
    onClose: onLogoutModalClose,
    onOpen: onLogoutOpen,
  } = useBoolean();

  const onClickBack = () => navigate(ROUTER.SETTINGS.MAIN);

  const openAuthThen = useCallback(
    (authType: TOKEN_TYPE, next: ChangeModalKey = null) => {
      const sensitiveForPasswordOnly = [
        TOKEN_TYPE.CHANGE_PASSWORD,
        TOKEN_TYPE.CHANGE_EMAIL,
        TOKEN_TYPE.TWO_FACTOR,
      ];

      if (usesOAuth && sensitiveForPasswordOnly.includes(authType)) {
        snack.error(t("user_messages.google_login_error"), {
          duration: 3000,
        });
        return;
      }

      setAuthModal({ open: true, authType, next });
    },
    [usesOAuth, t],
  );

  const closeAuth = () =>
    setAuthModal({ open: false, authType: undefined, next: null });

  const handleAuthenticated = () => {
    const next = authModal.next || null;
    closeAuth();
    setOpenModal(next);
  };

  const closeChangeModal = () => {
    setOpenModal(null);
  };

  const handleLogout = async () => {
    try {
      await logout().unwrap();
      clear("all");
      dispatch({ type: "RESET" });
      localStorage.clear();
      localStorage.setItem(LOCAL_STORAGE_KEYS.APP_THEME, theme);
      localStorage.setItem(LOCAL_STORAGE_KEYS.LANG, i18n.language);
      navigate(`${ROUTER.AUTH.LOGIN}?method=username`);
      snack.success(t("user_messages.logout_successfull"));
    } catch (error) {
      showResponseErrors(error);
    }
  };

  const handleDeactivateAccount = async () => {
    try {
      await deactivateAccount({
        token: authenticateToken as string,
      }).unwrap();

      localStorage.setItem(LOCAL_STORAGE_KEYS.APP_THEME, theme);
      localStorage.setItem(LOCAL_STORAGE_KEYS.LANG, i18n.language);
      clear("all");
      navigate(`${ROUTER.AUTH.LOGIN}?method=username`);
      snack.success(t("user_messages.account_deactivated"));
    } catch (error) {
      showResponseErrors(error);
    }
  };

  const renderPhoneNumber = () => {
    const country = COUNTRIES.find(
      (country) => country.code === user?.phoneNumber?.countryCode,
    );

    if (!country) return "";
    const formattedNumber = formatPhoneNumber(
      user?.phoneNumber?.number || "",
      country.format || "",
      country.code,
    );

    return formattedNumber;
  };

  const items = useMemo(
    () => [
      {
        id: "username",
        header: {
          icon: User,
          title: t("common.username"),
          subtitle: t("common.change_username"),
        },
        renderContent: () => (
          <AccountActionButton
            onClick={() => openAuthThen(TOKEN_TYPE.CHANGE_USERNAME, "username")}
          >
            <strong>@{user?.username}</strong>
          </AccountActionButton>
        ),
      },
      {
        id: "email",
        header: {
          icon: Mail,
          title: t("common.email_address"),
          subtitle: t("common.update_email"),
        },
        renderContent: () => (
          <AccountActionButton
            onClick={() => openAuthThen(TOKEN_TYPE.CHANGE_EMAIL, "email")}
          >
            {user?.email}
          </AccountActionButton>
        ),
      },
      {
        id: "password",
        header: {
          icon: Key,
          title: t("common.password"),
          subtitle: t("common.change_password"),
        },
        renderContent: () => (
          <AccountActionButton
            onClick={() => openAuthThen(TOKEN_TYPE.CHANGE_PASSWORD, "password")}
          >
            {"••••••••"}
          </AccountActionButton>
        ),
      },
      {
        id: "phone_number",
        header: {
          icon: Phone,
          title: t("common.phone_number"),
          subtitle: t("common.update_phone"),
        },
        renderContent: () => (
          <AccountActionButton
            onClick={() =>
              openAuthThen(TOKEN_TYPE.CHANGE_PHONE_NUMBER, "phone_number")
            }
          >
            {hasPhoneNumber
              ? renderPhoneNumber()
              : t("common.add_phone_number")}
          </AccountActionButton>
        ),
      },
      {
        id: "two_factor",
        header: {
          icon: ShieldCheck,
          title: t("common.two_factor_auth"),
          subtitle: t("common.manage_two_factor_auth"),
        },
        renderContent: () => (
          <AccountActionButton
            onClick={() => openAuthThen(TOKEN_TYPE.TWO_FACTOR, "two_factor")}
          >
            {isTwoFactorEnabled ? (
              <span className="text-success">{t(`common.enabled`)}</span>
            ) : (
              <span className="text-secondary">{t(`common.disabled`)}</span>
            )}
          </AccountActionButton>
        ),
      },
    ],
    [user, t],
  );

  useEffect(() => {
    const action = searchParams.get("action");
    const token = searchParams.get("token");

    if (!action || !token || action !== TOKEN_TYPE.CHANGE_EMAIL || !user)
      return;

    if (processedTokenRef.current === token) {
      return;
    }
    processedTokenRef.current = token;

    let decoded: Record<string, any> | null = null;
    try {
      decoded = jwtDecode<Record<string, any>>(token);
    } catch (err) {
      setSearchParams({});
      processedTokenRef.current = null;
      return;
    }

    if (!decoded || !decoded.userId || decoded.userId !== String(user._id)) {
      navigate(ROUTER.SETTINGS.ACCOUNT, { replace: true });
      setSearchParams({});
      processedTokenRef.current = null;
      return;
    }

    (async () => {
      try {
        onVerifiedOpen();

        await verifyChangeEmail({ token }).unwrap();
        snack.success(t("user_messages.email_changed_successfully"));
      } finally {
        setSearchParams({});
        processedTokenRef.current = null;
      }
    })();
  }, [searchParams, setSearchParams, navigate, user, t]);

  return (
    <Fragment>
      <section className="account-settings">
        <UniqueHeader
          headerTitle={t("common.account_settings")}
          headerSubtitle={t("common.manage_account_info")}
          onClickBack={onClickBack}
          isHeaderButtonDisabled={false}
        />
        <div className="account-settings-container">
          {LOADING_USER || LOADING_ACCOUNT || LOADING_GENERAL_SETTINGS ? (
            <SettingsSkeleton count={6} />
          ) : (
            <div className="account-settings-content">
              {items.map((it) => (
                <SettingCard key={it.id} header={it.header}>
                  {it.renderContent()}
                </SettingCard>
              ))}

              <SettingCard
                header={{
                  icon: Calendar,
                  title: t("common.account_info"),
                  subtitle: t("common.account_creation_login"),
                }}
              >
                <div className="account-info-grid">
                  <div className="account-info-item">
                    <p className="account-info-item-label">
                      {t("common.created_date")}
                    </p>
                    <p className="account-info-item-value">
                      <Calendar size={16} className="account-info-item-icon" />
                      {user?.createdAt ? DDMMMMYYYY(user.createdAt) : "-"}
                    </p>
                  </div>
                  <div className="account-info-item">
                    <p className="account-info-item-label">
                      {t("common.last_login")}
                    </p>
                    <p className="account-info-item-value">
                      <Clock size={16} className="account-info-item-icon" />
                      {showDateWithHour(
                        profile?.lastSeen as Date,
                        generalSettings?.timeZone.dateFormat as DATE_FORMAT,
                        generalSettings?.timeZone.timeFormat as TIME_FORMAT,
                        " - ",
                      )}
                    </p>
                  </div>
                </div>
              </SettingCard>

              <SettingCard
                header={{
                  icon: AlertTriangle,
                  title: t("common.danger_zone"),
                  subtitle: t("common.deactivate_or_delete"),
                  iconStyle: {
                    background:
                      "linear-gradient(135deg, #ef4444 0%, #dc2626 100%)",
                  },
                }}
              >
                <div className="account-danger-actions">
                  <Button
                    className="account-danger-button deactivate"
                    onClick={() =>
                      openAuthThen(
                        TOKEN_TYPE.DEACTIVATE_ACCOUNT,
                        "deactivate_account",
                      )
                    }
                  >
                    <span>
                      <LogOutIcon size={18} />
                      {t("common.deactivate_account")}
                    </span>
                    <ChevronRight size={18} />
                  </Button>
                  <Button
                    className="account-danger-button delete"
                    onClick={() =>
                      openAuthThen(TOKEN_TYPE.DELETE_ACCOUNT, "delete_account")
                    }
                  >
                    <span>
                      <Trash2 size={18} />
                      {t("common.delete_account")}
                    </span>
                    <ChevronRight size={18} />
                  </Button>
                  <Button
                    className="account-danger-button delete"
                    onClick={onLogoutOpen}
                  >
                    <span>
                      <LogOutIcon size={18} />
                      {t("common.logout_account")}
                    </span>
                    <ChevronRight size={18} />
                  </Button>
                </div>
              </SettingCard>
            </div>
          )}
        </div>
      </section>

      {authModal.open && (
        <AuthenticateModal
          open={authModal.open}
          onClose={closeAuth}
          onAuthenticate={handleAuthenticated}
          authType={authModal.authType as TOKEN_TYPE}
        />
      )}

      {openModal === "username" && (
        <UsernameModal open onClose={closeChangeModal} />
      )}
      {openModal === "email" && (
        <ChangeEmailModal open onClose={closeChangeModal} />
      )}
      {openModal === "password" && (
        <PasswordModal open onClose={closeChangeModal} />
      )}
      {openModal === "phone_number" && (
        <PhoneNumberModal open onClose={closeChangeModal} />
      )}
      {openModal === "two_factor" && (
        <TwoFactorModal
          open
          onClose={closeChangeModal}
          action={
            isTwoFactorEnabled
              ? TWO_FACTOR_ACTION.DISABLE
              : TWO_FACTOR_ACTION.ENABLE
          }
        />
      )}

      {verifiedModalOpen && (
        <VerifyChangeEmailModal
          open
          onClose={onVerifiedModalClose}
          isLoading={LOADING_VERIFY_CHANGE_EMAIL}
          isFailed={isVerifyChangeEmailFailed}
        />
      )}

      {openModal === "delete_account" && (
        <DeleteAccountModal open onClose={closeChangeModal} />
      )}

      {openModal === "deactivate_account" && (
        <ActionConfirmModal
          open
          onClose={closeChangeModal}
          onCancel={closeChangeModal}
          onConfirm={handleDeactivateAccount}
          header={{
            title: t("common.deactivate_account"),
          }}
          cancelBtn={{ title: t("common.cancel") }}
          confirmBtn={{
            title: t("common.confirm"),
            color: "var(--error-color)",
          }}
          icon={{
            content: TriangleAlert,
            color: "var(--error-color)",
          }}
          isLoading={LOADING_DEACTIVATE_ACCOUNT}
        >
          {t("common.deactivate_account_desc")}
        </ActionConfirmModal>
      )}

      {logoutModalOpen && (
        <ActionConfirmModal
          open={logoutModalOpen}
          onClose={onLogoutModalClose}
          onCancel={onLogoutModalClose}
          onConfirm={handleLogout}
          header={{
            title: t("common.logout"),
          }}
          cancelBtn={{ title: t("common.cancel") }}
          confirmBtn={{
            title: t("common.logout"),
            color: "var(--error-color)",
          }}
          isLoading={LOADING_LOGOUT}
        >
          {t("common.logout_desc")}
        </ActionConfirmModal>
      )}
    </Fragment>
  );
};

const AccountActionButton: FC<{
  onClick?: () => void;
  disabled?: boolean;
  right?: React.ReactNode;
  children?: React.ReactNode;
}> = ({ onClick, disabled, right, children }) => (
  <Button
    className="account-action-button"
    onClick={onClick}
    disabled={disabled}
    type="button"
  >
    <span>{children}</span>
    {right ?? <ChevronRight size={20} className="chevron" />}
  </Button>
);

export default AccountSettings;

```

## File: connectfy-client/src/modules/settings/AccountSettings/ui/components/Modal/UsernameModal/UsernameModal.tsx
```typescript
import { FC, useEffect } from "react";
import Modal from "@/components/Modal";
import { useTranslation } from "react-i18next";
import { IUpdateUsername } from "../../../../types/types";
import { checkEmptyString } from "@/common/utils/checkValues";
import { useFormik } from "formik";
import Input from "@/components/ui/CustomInput/Input/Input";
import { snack } from "@/common/utils/snackManager";
import { useUser } from "@/context/UserContext";
import { useUpdateUsernameMutation } from "@/modules/settings/AccountSettings/api/api";
import { useAuthStore } from "@/store/zustand/useAuthStore";
import { useErrors } from "@/hooks/useErrors";
import Button from "@/components/ui/CustomButton/Button/Button";
import useFormDisabled from "@/hooks/useFormDisabled";
import { CircleUser } from "lucide-react";

interface Props {
  open: boolean;
  onClose: () => void;
}

const UsernameModal: FC<Props> = ({ open, onClose }) => {
  const { t } = useTranslation();

  const { authenticateToken } = useAuthStore();
  const { showResponseErrors, showFormikErrors } = useErrors();

  const { user } = useUser();

  const [updateUsername, { isLoading: LOADING_UPDATE_USERNAME }] =
    useUpdateUsernameMutation();

  const initialState: IUpdateUsername = {
    username: null,
    token: authenticateToken as string,
  };

  const validate = ({ username }: IUpdateUsername): Record<string, any> => {
    const usernameRegex = /^[A-Za-z0-9._-]+$/;
    const errors: Record<string, any> = {};

    if (!username || !checkEmptyString(username))
      errors.username = t("error_messages.this_field_required");
    else if (!usernameRegex.test(username))
      errors.username = t("error_messages.invalid_username");
    else if (username.length < 3)
      errors.username = t("error_messages.min_length", {
        field: t("common.username"),
        length: 3,
      });

    return errors;
  };

  const formik = useFormik({
    initialValues: initialState,
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    validate: (values) => validate(values),
    onSubmit: async (values, { resetForm }) => {
      try {
        await updateUsername(values).unwrap();
        snack.success(t("user_messages.username_changed_successfully"));
        resetForm();
        onClose();
      } catch (error) {
        showResponseErrors(error);
      }
    },
  });

  const isDisabled = useFormDisabled<IUpdateUsername>({
    formik,
    loading: LOADING_UPDATE_USERNAME,
    validationRules: [
      (values) => !!values.username,
      (values) => !!values.username && checkEmptyString(values.username),
      (values) => !!values.username && values.username.length >= 3,
      (values) => !!values.username && values.username.length <= 30,
      (values) => !!values.username && values.username !== user?.username,
    ],
  });

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (isDisabled) return;
    const errors = await formik.validateForm();
    if (Object.keys(errors).length > 0) {
      showFormikErrors(errors);
      return;
    }
    formik.handleSubmit(e as any);
  };

  const handleClose = () => {
    formik.resetForm();
    onClose();
  };

  useEffect(() => {
    if (!open) return;

    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === "Enter") {
        e.preventDefault();
        handleSubmit(e as any);
      } else if (e.key === "Escape") {
        e.preventDefault();
        handleClose();
      }
    };

    document.addEventListener("keydown", handleKeyDown);
    return () => document.removeEventListener("keydown", handleKeyDown);
  }, [open, formik]);

  return (
    <Modal
      open={open}
      onClose={handleClose}
      onMouseDown={(e) => e.target === e.currentTarget && handleClose()}
    >
      <div className="bg-(--auth-main-bg) rounded-2xl p-6 sm:p-8 max-w-[480px] w-full shadow-(--card-shadow) animate-fade-in mx-auto overflow-hidden">
        {/* Header */}
        <div className="mb-7">
          <h2 className="text-2xl font-bold text-(--text-primary) m-0">
            {t("common.change_username")}
          </h2>
          <p className="text-sm text-(--muted-color) leading-relaxed">
            {t("common.update_username_description")}
          </p>
        </div>

        {/* Form */}
        <form onSubmit={handleSubmit} className="flex flex-col gap-6">
          <div className="flex flex-col">
            <Input
              inputSize="large"
              id="username"
              name="username"
              title={t("common.username")}
              value={formik.values.username || ""}
              onChange={(e) => {
                const value = e.target.value || null;
                if (value && value.length > 30) return;
                formik.setFieldValue("username", value);
              }}
              onBlur={formik.handleBlur}
              isError={!!formik.errors.username}
              disabled={LOADING_UPDATE_USERNAME}
              autoFocus
              autoComplete="off"
              icon={<CircleUser />}
            />
          </div>

          {/* Actions */}
          <div className="flex gap-3 mt-2">
            <Button
              type="button"
              onClick={handleClose}
              className="flex-1 px-6 py-3 rounded-xl text-[15px] font-semibold transition-all duration-200 bg-(--input-bg) text-(--text-primary) border border-(--input-border) hover:bg-(--input-border)"
              disabled={LOADING_UPDATE_USERNAME}
              title={t("common.cancel")}
            />
            <Button
              type="submit"
              className="flex-1 px-6 py-3 rounded-xl text-[15px] font-semibold transition-all duration-200 bg-(--primary-color) text-white shadow-(--shadow-color) hover:shadow-(--active-shadow) flex items-center justify-center min-h-[48px]"
              disabled={isDisabled}
              isLoading={LOADING_UPDATE_USERNAME}
              title={t("common.save")}
            />
          </div>
        </form>
      </div>
    </Modal>
  );
};

export default UsernameModal;

```

## File: connectfy-client/src/modules/settings/AccountSettings/ui/components/Modal/DeleteAccountModal/DeleteAccountModal.tsx
```typescript
import { FC } from "react";
import Modal from "@/components/Modal";
import { useTranslation } from "react-i18next";
import { DELETE_REASON_CODE } from "@/common/enums/enums";
import { useFormik } from "formik";
import { checkEmptyString } from "@/common/utils/checkValues";
import { TriangleAlert } from "lucide-react";
import { ROUTER } from "@/common/constants/routet";
import { snack } from "@/common/utils/snackManager";
import { IDeleteAccount } from "../../../../types/types";
import { useDeleteAccountMutation } from "@/modules/settings/AccountSettings/api/api";
import { useAuthStore } from "@/store/zustand/useAuthStore";
import { useErrors } from "@/hooks/useErrors";
import Button from "@/components/ui/CustomButton/Button/Button";
import Textarea from "@/components/ui/CustomTextArea/TextArea/Textarea";
import { useAppNavigation } from "@/hooks/useAppNavigation";

interface Props {
  open: boolean;
  onClose: () => void;
}

const MAX_REASON_LENGTH = 200;

const DeleteAccountModal: FC<Props> = ({ open, onClose }) => {
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();
  const { showResponseErrors } = useErrors();
  const { authenticateToken, clear } = useAuthStore();
  const [deleteAccount, { isLoading }] = useDeleteAccountMutation();

  const initialState: IDeleteAccount = {
    token: authenticateToken as string,
    reasonCode: null,
    reasonDescription: null,
  };

  const validate = (values: IDeleteAccount) => {
    const errors: Record<string, string> = {};
    if (!values.reasonCode || !checkEmptyString(values.reasonCode)) {
      errors.reasonCode = t("error_messages.this_field_required");
    }
    return errors;
  };

  const formik = useFormik({
    initialValues: initialState,
    validate,
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    onSubmit: async (values, { resetForm }) => {
      try {
        await deleteAccount(values).unwrap();
        resetForm();
        onClose();
        clear("all");
        navigate(`${ROUTER.AUTH.LOGIN}?method=username`);
        snack.success(t("user_messages.account_deleted"));
      } catch (error) {
        showResponseErrors(error);
      }
    },
  });

  const reasons = [
    { value: null, label: t("common.selectReason") },
    {
      value: DELETE_REASON_CODE.NOT_USEFUL,
      label: t(`enum.${DELETE_REASON_CODE.NOT_USEFUL}`),
    },
    {
      value: DELETE_REASON_CODE.PRIVACY_CONCERNS,
      label: t(`enum.${DELETE_REASON_CODE.PRIVACY_CONCERNS}`),
    },
    {
      value: DELETE_REASON_CODE.FOUND_ALTERNATIVE,
      label: t(`enum.${DELETE_REASON_CODE.FOUND_ALTERNATIVE}`),
    },
    {
      value: DELETE_REASON_CODE.TECHNICAL_ISSUES,
      label: t(`enum.${DELETE_REASON_CODE.TECHNICAL_ISSUES}`),
    },
    {
      value: DELETE_REASON_CODE.OTHER,
      label: t(`enum.${DELETE_REASON_CODE.OTHER}`),
    },
  ];

  return (
    <Modal open={open} onClose={onClose}>
      <div className="w-full max-w-[440px] overflow-hidden rounded-2xl bg-(--card-bg) p-6 shadow-2xl animate-in fade-in zoom-in-95 duration-200">
        {/* Header Section */}
        <div className="flex flex-col items-center text-center">
          <div className="mb-4 flex size-14 items-center justify-center rounded-full bg-red-500/10 text-red-600">
            <TriangleAlert size={30} strokeWidth={2.5} />
          </div>

          <h2 className="mb-2 text-2xl font-bold tracking-tight text-(--text-color)">
            {t("common.delete_account_title")}
          </h2>

          <p className="mb-8 text-sm leading-relaxed text-(--muted-color)">
            {t("common.delete_account_description")}
          </p>
        </div>

        {/* Form Section */}
        <form onSubmit={formik.handleSubmit} className="space-y-6">
          <div className="space-y-2">
            <label
              htmlFor="reasonCode"
              className="mb-2 block text-sm font-semibold text-(--text-color)"
            >
              {t("common.delete_account_reason_label")}
            </label>

            <div className="relative group">
              <select
                id="reasonCode"
                name="reasonCode"
                value={formik.values.reasonCode || ""}
                onChange={formik.handleChange}
                onBlur={formik.handleBlur}
                className="w-full appearance-none rounded-xl border bg-(--secondary-color) px-4 py-3 text-sm text-(--text-color) outline-none transition-all placeholder:text-muted-foreground focus:ring-2 focus:ring-red-500/20 border-(--border-color) hover:border-(--muted-color"
              >
                {reasons.map((reason) => (
                  <option
                    key={reason.value}
                    value={reason.value || ""}
                    className="bg-(--card-bg)"
                  >
                    {reason.label}
                  </option>
                ))}
              </select>
            </div>

            {formik.values.reasonCode &&
              formik.values.reasonCode !==
                DELETE_REASON_CODE.FOUND_ALTERNATIVE && (
                <div className="relative mt-4">
                  <Textarea
                    id="reasonDescription"
                    name="reasonDescription"
                    value={formik.values.reasonDescription || ""}
                    onChange={formik.handleChange}
                    onBlur={formik.handleBlur}
                    rows={4}
                    maxLength={MAX_REASON_LENGTH}
                    isFloating={false}
                    placeholder={t("common.reason_description")}
                    className="w-full! rounded-xl! border-(--border-color)! bg-(--secondary-color)! text-sm! focus:border-red-500/50! focus:ring-red-500/20! max-h-[130px] resize-none!"
                  />

                  {/* Character Limit Counter */}
                  <div className="mt-1 flex justify-end">
                    <span
                      className={`text-[11px] font-medium ${(formik.values.reasonDescription?.length || 0) >= MAX_REASON_LENGTH ? "text-red-500" : "text-(--muted-color)"}`}
                    >
                      {formik.values.reasonDescription?.length || 0} /{" "}
                      {MAX_REASON_LENGTH}
                    </span>
                  </div>
                </div>
              )}
          </div>

          {/* Action Buttons */}
          <div className="flex flex-col-reverse gap-3 sm:flex-row sm:pt-2">
            <Button
              type="button"
              onClick={onClose}
              disabled={isLoading}
              className="flex-1 rounded-xl border border-(--border-color) bg-transparent py-3 text-sm font-semibold text-(--text-color) transition-all hover:bg-(--disabled-bg)"
              title={t("common.cancel")}
            />
            <Button
              type="submit"
              disabled={
                formik.isSubmitting ||
                isLoading ||
                !formik.values.reasonCode ||
                !checkEmptyString(formik.values.reasonCode)
              }
              isLoading={formik.isSubmitting || isLoading}
              className="flex-1 rounded-xl bg-(--error-color) py-3 text-sm font-semibold text-white shadow-lg shadow-red-600/20 transition-all duration-300"
              title={
                formik.isSubmitting || isLoading
                  ? t("common.deleting")
                  : t("common.delete_account_confirm")
              }
            />
          </div>
        </form>
      </div>
    </Modal>
  );
};

export default DeleteAccountModal;

```

## File: connectfy-client/src/modules/settings/AccountSettings/ui/components/Modal/TwoFactorModal/TwoFactorModal.tsx
```typescript
import { FC, useEffect } from "react";
import Modal from "@/components/Modal";
import { useTranslation } from "react-i18next";
import { IUpdateTwoFactor } from "../../../../types/types";
import { useFormik } from "formik";
import { snack } from "@/common/utils/snackManager";
import { useUpdateTwoFactorMutation } from "@/modules/settings/AccountSettings/api/api";
import { useAuthStore } from "@/store/zustand/useAuthStore";
import { useErrors } from "@/hooks/useErrors";
import Button from "@/components/ui/CustomButton/Button/Button";
import useFormDisabled from "@/hooks/useFormDisabled";
import { TWO_FACTOR_ACTION } from "@/common/enums/enums";

interface Props {
  open: boolean;
  onClose: () => void;
  action: TWO_FACTOR_ACTION;
}

const TwoFactorModal: FC<Props> = ({ open, onClose, action }) => {
  const { t } = useTranslation();
  const { authenticateToken } = useAuthStore();
  const { showResponseErrors, showFormikErrors } = useErrors();

  const [updateTwoFactor, { isLoading }] = useUpdateTwoFactorMutation();

  const initialState: IUpdateTwoFactor = {
    action,
    token: authenticateToken as string,
  };

  const formik = useFormik({
    initialValues: initialState,
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    onSubmit: async (values, { resetForm }) => {
      try {
        await updateTwoFactor(values).unwrap();
        snack.success(t("user_messages.information_updated"));
        resetForm();
        onClose();
      } catch (error) {
        showResponseErrors(error);
      }
    },
  });

  const isDisabled = useFormDisabled<IUpdateTwoFactor>({
    formik,
    loading: isLoading,
    validationRules: [],
  });

  const handleOverlayPointerDown = (e: React.MouseEvent<HTMLDivElement>) => {
    if (e.target === e.currentTarget) onClose();
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    const errors = await formik.validateForm();
    if (Object.keys(errors).length > 0) {
      showFormikErrors(errors);
      return;
    }
    formik.handleSubmit(e as any);
  };

  useEffect(() => {
    if (!open) return;

    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === "Enter") {
        e.preventDefault();
        if (isDisabled) return;
        handleSubmit(e as any);
      }
      if (e.key === "Escape") {
        if (isLoading) return;
        onClose();
      }
    };

    document.addEventListener("keydown", handleKeyDown);
    return () => document.removeEventListener("keydown", handleKeyDown);
  }, [open, formik]);

  return (
    <Modal open={open} onClose={onClose} onMouseDown={handleOverlayPointerDown}>
      <div className="bg-(--auth-main-bg) rounded-2xl p-6 sm:p-8 max-w-[480px] w-full shadow-(--card-shadow) animate-fade-in mx-auto">
        {/* Header */}
        <div className="mb-7">
          <h2 className="text-2xl font-semibold text-(--text-primary) mb-2 leading-tight">
            {action === TWO_FACTOR_ACTION.ENABLE
              ? t("common.enable_two_factor_authentication")
              : t("common.disable_two_factor_authentication")}
          </h2>

          <p className="text-sm text-(--muted-color) leading-relaxed">
            {action === TWO_FACTOR_ACTION.ENABLE
              ? t("common.enable_two_factor_authentication_description")
              : t("common.disable_two_factor_authentication_description")}
          </p>
        </div>

        {/* Form */}
        <form className="flex flex-col gap-6" onSubmit={handleSubmit}>
          {/* Actions */}
          <div className="flex gap-3 mt-2">
            <Button
              type="button"
              onClick={onClose}
              className="flex-1 px-6 py-3 rounded-[10px] text-[15px] font-medium transition-all duration-200 bg-(--input-bg) text-(--text-primary) border border-(--input-border) hover:bg-(--input-border)"
              disabled={isLoading}
              title={t("common.cancel")}
            />
            <Button
              type="submit"
              className={`flex-1 px-6 py-3 rounded-[10px] text-[15px] font-medium transition-all duration-200 ${action === TWO_FACTOR_ACTION.ENABLE ? "bg-(--primary-color)" : "bg-(--error-color)"} text-white shadow-(--shadow-color) hover:shadow-(--active-shadow) flex items-center justify-center gap-2`}
              disabled={isDisabled}
              isLoading={isLoading}
              title={
                action === TWO_FACTOR_ACTION.ENABLE
                  ? t("common.enable")
                  : t("common.disable")
              }
            />
          </div>
        </form>
      </div>
    </Modal>
  );
};

export default TwoFactorModal;

```

## File: connectfy-client/src/modules/settings/AccountSettings/ui/components/Modal/EmailModal/ChangeEmailModal/ChangeEmailModal.tsx
```typescript
import { FC, Fragment, useEffect, useState } from "react";
import Modal from "@/components/Modal";
import { useTranslation } from "react-i18next";
import { IUpdateEmail } from "../../../../../types/types";
import { checkEmptyString } from "@/common/utils/checkValues";
import { useFormik } from "formik";
import { useUpdateEmailMutation } from "@/modules/settings/AccountSettings/api/api";
import Input from "@/components/ui/CustomInput/Input/Input";
import { useUser } from "@/context/UserContext";
import { useAuthStore } from "@/store/zustand/useAuthStore";
import { useErrors } from "@/hooks/useErrors";
import Button from "@/components/ui/CustomButton/Button/Button";
import useFormDisabled from "@/hooks/useFormDisabled";
import { Mail, MailCheck } from "lucide-react";

interface Props {
  open: boolean;
  onClose: () => void;
}

const ChangeEmailModal: FC<Props> = ({ open, onClose }) => {
  const { t } = useTranslation();

  const { authenticateToken } = useAuthStore();

  const { showResponseErrors, showFormikErrors } = useErrors();

  const { user } = useUser();

  const [updateEmail, { isLoading }] = useUpdateEmailMutation();

  const [success, setSuccess] = useState<boolean>(false);
  const [email, setEmail] = useState<string | null>(null);

  const initialState: IUpdateEmail = {
    email: null,
    token: authenticateToken as string,
  };

  const validate = ({ email }: IUpdateEmail): Record<string, any> => {
    const errors: Record<string, any> = {};

    if (!email || !checkEmptyString(email)) {
      errors.email = t("error_messages.this_field_required");
      return errors;
    }

    const value = email.trim();

    if (value.length > 254) {
      errors.email = t("error_messages.email_too_long");
      return errors;
    }

    const atCount = (value.match(/@/g) || []).length;
    if (atCount !== 1) {
      errors.email = t("error_messages.invalid_email_format");
      return errors;
    }

    const [localPart, domainPart] = value.split("@");

    if (!localPart || !domainPart) {
      errors.email = t("error_messages.invalid_email_format");
      return errors;
    }

    if (localPart.length > 64) {
      errors.email = t("error_messages.email_local_part_too_long");
      return errors;
    }

    if (/\.\./.test(localPart) || /\.\./.test(domainPart)) {
      errors.email = t("error_messages.email_consecutive_dots");
      return errors;
    }

    const localAllowedRe = /^[A-Za-z0-9!#$%&'*+/=?^_`{|}~.-]+$/;
    if (!localAllowedRe.test(localPart)) {
      errors.email = t("error_messages.email_invalid_local_chars");
      return errors;
    }

    if (localPart.startsWith(".") || localPart.endsWith(".")) {
      errors.email = t("error_messages.invalid_email_format");
      return errors;
    }

    const domainLabels = domainPart.split(".");
    if (domainLabels.some((label) => label.length === 0)) {
      errors.email = t("error_messages.email_invalid_domain");
      return errors;
    }

    const domainLabelRe = /^[A-Za-z0-9-]+$/;
    for (const label of domainLabels) {
      if (label.length < 1 || label.length > 63) {
        errors.email = t("error_messages.email_invalid_domain");
        return errors;
      }
      if (!domainLabelRe.test(label)) {
        errors.email = t("error_messages.email_invalid_domain");
        return errors;
      }
      if (label.startsWith("-") || label.endsWith("-")) {
        errors.email = t("error_messages.email_invalid_domain");
        return errors;
      }
    }

    const tld = domainLabels[domainLabels.length - 1];
    if (!/^[A-Za-z]{2,}$/.test(tld)) {
      errors.email = t("error_messages.email_invalid_domain");
      return errors;
    }

    const simpleEmailRe = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!simpleEmailRe.test(value)) {
      errors.email = t("error_messages.invalid_email_format");
      return errors;
    }

    return errors;
  };

  const formik = useFormik({
    initialValues: initialState,
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    validate: (values) => validate(values),
    onSubmit: async (values, { resetForm }) => {
      try {
        await updateEmail(values).unwrap();
        resetForm();
        setSuccess(true);
        setEmail(values.email);
      } catch (error) {
        showResponseErrors(error);
      }
    },
  });

  const isDisabled = useFormDisabled<IUpdateEmail>({
    formik,
    loading: isLoading,
    validationRules: [
      (values) => !!values.email,
      (values) => values.email !== user?.email,
    ],
  });

  const handleOverlayPointerDown = (e: React.MouseEvent<HTMLDivElement>) => {
    if (e.target === e.currentTarget) {
      onClose();
    }
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    const errors = await formik.validateForm();
    if (Object.keys(errors).length > 0) {
      showFormikErrors(errors);
      return;
    }
    formik.handleSubmit(e as any);
  };

  const handleClose = () => {
    setSuccess(false);
    setEmail(null);
    formik.resetForm();
    onClose();
  };

  useEffect(() => {
    if (!open) return;

    const handleGlobalKeyDown = (e: KeyboardEvent) => {
      if (e.key === "Enter") {
        e.preventDefault();
        if (isLoading || !formik.values.email) return;
        formik.submitForm();
      }

      if (e.key === "Escape") {
        e.preventDefault();
        if (isLoading) return;
        onClose();
      }
    };

    document.addEventListener("keydown", handleGlobalKeyDown);
    return () => {
      document.removeEventListener("keydown", handleGlobalKeyDown);
    };
  }, [open, formik]);

  return (
    <Modal
      open={open}
      onClose={handleClose}
      onMouseDown={handleOverlayPointerDown}
    >
      <div className="bg-(--auth-main-bg) rounded-2xl p-6 sm:p-8 max-w-[480px] w-full shadow-(--card-shadow) animate-fade-in mx-auto overflow-hidden">
        {!success ? (
          <Fragment>
            {/* Header */}
            <div className="mb-7">
              <h2 className="text-2xl font-semibold text-(--text-primary) mb-2 leading-tight">
                {t("common.update_email")}
              </h2>
              <p className="text-sm text-(--muted-color) leading-relaxed">
                {t("common.update_email_description")}
              </p>
            </div>

            {/* Form */}
            <form onSubmit={handleSubmit} className="flex flex-col gap-6">
              <div className="flex flex-col">
                <Input
                  inputSize="large"
                  id="email"
                  type="email"
                  name="email"
                  title={t("common.email")}
                  value={formik.values.email || ""}
                  onChange={(e) => {
                    const value = e.target.value || null;
                    if (value && value.length > 254) return;
                    formik.setFieldValue("email", value);
                  }}
                  onBlur={formik.handleBlur}
                  isError={!!formik.errors.email}
                  disabled={isLoading}
                  autoFocus
                  autoComplete="off"
                  icon={<Mail />}
                />
              </div>

              {/* Actions */}
              <div className="flex gap-3 mt-2">
                <Button
                  type="button"
                  onClick={handleClose}
                  className="flex-1 px-6 py-3 rounded-[10px] text-[15px] font-medium transition-all duration-200 bg-(--input-bg) text-(--text-primary) border border-(--input-border) hover:bg-(--input-border)"
                  disabled={isLoading}
                  title={t("common.cancel")}
                />
                <Button
                  type="submit"
                  className="flex-1 px-6 py-3 rounded-[10px] text-[15px] font-medium transition-all duration-200 bg-(--primary-color) text-white shadow-(--shadow-color) hover:shadow-(--active-shadow) flex items-center justify-center gap-2"
                  disabled={isDisabled}
                  isLoading={isLoading}
                  title={t("common.save")}
                />
              </div>
            </form>
          </Fragment>
        ) : (
          /* Success State */
          <div className="flex flex-col items-center text-center animate-fade-in py-4">
            <div className="w-[72px] h-[72px] bg-[rgba(0,0,0,0.03)] dark:bg-[rgba(255,255,255,0.03)] rounded-full flex items-center justify-center mb-4 animate-bounce-custom">
              <MailCheck size={40} className="text-(--primary-color)" />
            </div>

            <h3 className="text-lg font-semibold text-(--text-primary) mb-2.5 m-0">
              {t("user_messages.email_confirmation_sent") ||
                "Confirmation email sent"}
            </h3>

            <p className="text-[14px] text-(--muted-color) leading-[1.4] mb-6 max-w-[420px] m-0">
              {email
                ? t("user_messages.check_inbox_for_confirmation", { email })
                : t("user_messages.check_inbox")}
            </p>

            <Button
              type="button"
              onClick={handleClose}
              className="w-full sm:w-auto px-8 py-3 rounded-[10px] text-[15px] font-medium transition-all duration-200 bg-(--primary-color) text-white hover:shadow-(--active-shadow)"
              title={t("common.ok")}
            />
          </div>
        )}
      </div>
    </Modal>
  );
};

export default ChangeEmailModal;

```

## File: connectfy-client/src/modules/settings/AccountSettings/ui/components/Modal/EmailModal/VerifyChangeEmailModal/VerifyChangeEmailModal.tsx
```typescript
import Modal from "@/components/Modal";
import Button from "@/components/ui/CustomButton/Button/Button";
import { CheckCircle } from "lucide-react";
import { FC, Fragment } from "react";
import { useTranslation } from "react-i18next";

interface Props {
  open: boolean;
  onClose: () => void;
  isLoading: boolean;
  isFailed: boolean;
}

const VerifyChangeEmailModal: FC<Props> = ({
  open,
  onClose,
  isLoading,
  isFailed,
}) => {
  const { t } = useTranslation();

  if (!isLoading && isFailed) {
    onClose();
    return;
  }

  return (
    <Modal open={open} onClose={onClose}>
      <div className="bg-(--auth-main-bg) rounded-2xl p-6 sm:p-8 max-w-[480px] w-full shadow-(--card-shadow) animate-fade-in mx-auto overflow-hidden">
        <div className="flex flex-col items-center text-center animate-fade-in py-4">
          {isLoading ? (
            <Fragment>
              {/* Verifying Loading Illustration */}
              <div
                className="relative w-20 h-20 bg-[rgba(0,0,0,0.03)] dark:bg-[rgba(255,255,255,0.03)] rounded-full flex items-center justify-center mb-6"
                aria-hidden
              >
                {/* Spinner */}
                <div
                  className="w-10 h-10 border-4 border-black/10 dark:border-white/10 border-t-(--primary-color) rounded-full animate-spin"
                  aria-hidden
                />

                {/* Badge Icon */}
                <span
                  className="absolute -right-1.5 -bottom-1.5 w-8 h-8 rounded-full flex items-center justify-center bg-(--auth-main-bg) border border-(--input-border) shadow-md"
                  aria-hidden
                >
                  <svg
                    width="16"
                    height="16"
                    viewBox="0 0 24 24"
                    fill="none"
                    xmlns="http://www.w3.org/2000/svg"
                  >
                    <path
                      d="M12 7v6l4 2"
                      stroke="var(--primary-color)"
                      strokeWidth="2"
                      strokeLinecap="round"
                      strokeLinejoin="round"
                    />
                    <circle
                      cx="12"
                      cy="12"
                      r="8"
                      stroke="var(--primary-color)"
                      strokeWidth="1.8"
                    />
                  </svg>
                </span>
              </div>

              {/* Texts */}
              <div className="mb-2">
                <h3 className="text-xl font-bold text-(--text-primary) mb-3 m-0">
                  {t("common.verifying_email") || "Verifying..."}
                </h3>
                <p className="text-[15px] text-(--muted-color) leading-relaxed max-w-[340px] m-0 mx-auto">
                  {t("common.verifying_email_sub") ||
                    "We are verifying your new email address. This may take a few seconds."}
                </p>
              </div>
            </Fragment>
          ) : (
            <Fragment>
              {/* Success Illustration */}
              {!isFailed && (
                <Fragment>
                  <div
                    className="w-20 h-20 bg-[rgba(0,0,0,0.03)] dark:bg-[rgba(255,255,255,0.03)] rounded-full flex items-center justify-center mb-6 animate-bounce-custom"
                    aria-hidden
                  >
                    <CheckCircle size={40} className="text-(--primary-color)" />
                  </div>

                  {/* Texts */}
                  <div className="mb-8">
                    <h3 className="text-xl font-bold text-(--text-primary) m-0">
                      {t("user_messages.email_changed_successfully")}
                    </h3>
                  </div>

                  {/* Actions */}
                  <div className="w-full flex justify-center">
                    <Button
                      type="button"
                      onClick={onClose}
                      className="w-full sm:w-2/3 px-6 py-3 rounded-xl text-[15px] font-semibold transition-all duration-200 bg-(--input-bg) text-(--text-primary) border border-(--input-border) hover:bg-(--input-border)"
                      title={t("common.close")}
                    />
                  </div>
                </Fragment>
              )}
            </Fragment>
          )}
        </div>
      </div>
    </Modal>
  );
};

export default VerifyChangeEmailModal;

```

## File: connectfy-client/src/modules/settings/AccountSettings/ui/components/Modal/PasswordModal/PasswordModal.tsx
```typescript
import { FC, useEffect } from "react";
import Modal from "@/components/Modal";
import { useTranslation } from "react-i18next";
import { IUpdatePassword } from "../../../../types/types";
import { checkEmptyString } from "@/common/utils/checkValues";
import { useFormik } from "formik";
import { snack } from "@/common/utils/snackManager";
import PasswordInput from "@/components/ui/CustomInput/PasswordInput/PasswordInput";
import { useUpdatePasswordMutation } from "@/modules/settings/AccountSettings/api/api";
import { useAuthStore } from "@/store/zustand/useAuthStore";
import { useErrors } from "@/hooks/useErrors";
import Button from "@/components/ui/CustomButton/Button/Button";
import useFormDisabled from "@/hooks/useFormDisabled";

interface Props {
  open: boolean;
  onClose: () => void;
}

const PasswordModal: FC<Props> = ({ open, onClose }) => {
  const { t } = useTranslation();
  const { authenticateToken } = useAuthStore();
  const { showResponseErrors, showFormikErrors } = useErrors();

  const [updatePassword, { isLoading }] = useUpdatePasswordMutation();

  const initialState: IUpdatePassword = {
    password: null,
    confirmPassword: null,
    token: authenticateToken as string,
  };

  const validate = ({
    password,
    confirmPassword,
  }: IUpdatePassword): Record<string, any> => {
    const errors: Record<string, any> = {};
    const passwordComplexityRegex =
      /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^A-Za-z0-9]).+$/;

    if (!password || !checkEmptyString(password))
      errors.password = t("error_messages.password_is_required");
    else if (password.length < 8 || !passwordComplexityRegex.test(password))
      errors.password = t("error_messages.password_rule");

    if (password !== confirmPassword)
      errors.confirmPassword = t("error_messages.password_mismatch");

    return errors;
  };

  const formik = useFormik({
    initialValues: initialState,
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    validate: (values) => validate(values),
    onSubmit: async (values, { resetForm }) => {
      try {
        await updatePassword(values).unwrap();
        snack.success(t("user_messages.password_changed_successfully"));
        resetForm();
        onClose();
      } catch (error) {
        showResponseErrors(error);
      }
    },
  });

  const isDisabled = useFormDisabled<IUpdatePassword>({
    formik,
    loading: isLoading,
    validationRules: [
      (values) => !!values.password,
      (values) => !!values.confirmPassword,
      (values) => values.password === values.confirmPassword,
    ],
  });

  const handleOverlayPointerDown = (e: React.MouseEvent<HTMLDivElement>) => {
    if (e.target === e.currentTarget) onClose();
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    const errors = await formik.validateForm();
    if (Object.keys(errors).length > 0) {
      showFormikErrors(errors);
      return;
    }
    formik.handleSubmit(e as any);
  };

  useEffect(() => {
    if (!open) return;

    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === "Enter") {
        e.preventDefault();
        if (isDisabled) return;
        handleSubmit(e as any);
      }
      if (e.key === "Escape") {
        if (isLoading) return;
        onClose();
      }
    };

    document.addEventListener("keydown", handleKeyDown);
    return () => document.removeEventListener("keydown", handleKeyDown);
  }, [open, formik]);

  return (
    <Modal open={open} onClose={onClose} onMouseDown={handleOverlayPointerDown}>
      <div className="bg-(--auth-main-bg) rounded-2xl p-6 sm:p-8 max-w-[480px] w-full shadow-(--card-shadow) animate-fade-in mx-auto">
        {/* Header */}
        <div className="mb-7">
          <h2 className="text-2xl font-semibold text-(--text-primary) mb-2 leading-tight">
            {t("common.change_password")}
          </h2>
          <p className="text-sm text-(--muted-color) leading-relaxed">
            {t("common.change_password_description")}
          </p>
        </div>

        {/* Form */}
        <form className="flex flex-col gap-6" onSubmit={handleSubmit}>
          <div className="flex flex-col gap-4">
            {/* Password Field */}
            <div className="flex flex-col">
              <PasswordInput
                inputSize="large"
                id="password"
                name="password"
                title={t("common.password")}
                value={formik.values.password || ""}
                onChange={(e) => {
                  const val = e.target.value;
                  if (val.length <= 30)
                    formik.setFieldValue("password", val || null);
                }}
                onBlur={formik.handleBlur}
                isError={!!formik.errors.password}
                disabled={isLoading}
                autoFocus
                autoComplete="off"
                showGenerateButton
                onGenerate={(value?: string) => {
                  formik.setFieldValue("password", value);
                  formik.setFieldValue("confirmPassword", value);
                }}
              />
            </div>

            {/* Confirm Password Field */}
            <div className="flex flex-col">
              <PasswordInput
                inputSize="large"
                id="confirmPassword"
                name="confirmPassword"
                title={t("common.confirm_password")}
                value={formik.values.confirmPassword || ""}
                onChange={(e) => {
                  const val = e.target.value;
                  if (val.length <= 30)
                    formik.setFieldValue("confirmPassword", val || null);
                }}
                onBlur={formik.handleBlur}
                isError={!!formik.errors.confirmPassword}
                disabled={isLoading}
                autoComplete="off"
              />
            </div>
          </div>

          {/* Actions */}
          <div className="flex gap-3 mt-2">
            <Button
              type="button"
              onClick={onClose}
              className="flex-1 px-6 py-3 rounded-[10px] text-[15px] font-medium transition-all duration-200 bg-(--input-bg) text-(--text-primary) border border-(--input-border) hover:bg-(--input-border)"
              disabled={isLoading}
              title={t("common.cancel")}
            />
            <Button
              type="submit"
              className="flex-1 px-6 py-3 rounded-[10px] text-[15px] font-medium transition-all duration-200 bg-(--primary-color) text-white shadow-(--shadow-color) hover:shadow-(--active-shadow) flex items-center justify-center gap-2"
              disabled={isDisabled}
              isLoading={isLoading}
              title={t("common.save")}
            />
          </div>
        </form>
      </div>
    </Modal>
  );
};

export default PasswordModal;

```

## File: connectfy-client/src/modules/settings/AccountSettings/ui/components/Modal/PhoneNumberModal/PhoneNumberModal.tsx
```typescript
import { FC, useEffect, useState } from "react";
import Modal from "@/components/Modal";
import { useTranslation } from "react-i18next";
import { IUpdatePhoneNumber } from "../../../../types/types";
import { useFormik } from "formik";
import { ModalView, PHONE_NUMBER_ACTION } from "@/common/enums/enums";
import PhoneNumberForm from "@/components/Form/PhoneNumberForm/PhoneNumberForm";
import { checkEmptyString } from "@/common/utils/checkValues";
import { snack } from "@/common/utils/snackManager";
import { COUNTRIES } from "@/common/constants/constants";
import { ChevronRight, Pencil, Trash } from "lucide-react";
import { useUser } from "@/context/UserContext";
import { useUpdatePhoneNumberMutation } from "@/modules/settings/AccountSettings/api/api";
import { useAuthStore } from "@/store/zustand/useAuthStore";
import { useErrors } from "@/hooks/useErrors";
import Button from "@/components/ui/CustomButton/Button/Button";

interface Props {
  open: boolean;
  onClose: () => void;
}

const PhoneNumberModal: FC<Props> = ({ open, onClose }) => {
  const { t } = useTranslation();
  const { authenticateToken } = useAuthStore();
  const { showResponseErrors } = useErrors();

  const { user } = useUser();
  const [updatePhoneNumber, { isLoading }] = useUpdatePhoneNumberMutation();

  const hasPhoneNumber = !!user?.phoneNumber;
  const [view, setView] = useState<ModalView>(
    hasPhoneNumber ? ModalView.SELECTION : ModalView.FORM,
  );

  const initialState: IUpdatePhoneNumber = {
    action: user?.phoneNumber ? null : PHONE_NUMBER_ACTION.UPDATE,
    phoneNumber: user?.phoneNumber || null,
    token: authenticateToken as string,
  };

  const validate = ({
    action,
    phoneNumber,
  }: IUpdatePhoneNumber): Record<string, any> => {
    const errors: Record<string, any> = {};
    if (action === PHONE_NUMBER_ACTION.UPDATE) {
      if (
        !phoneNumber?.fullPhoneNumber ||
        !checkEmptyString(phoneNumber.fullPhoneNumber)
      ) {
        errors.phoneNumber = t("error_messages.this_field_required");
      }
    }
    const country = COUNTRIES.find((c) => c.code === phoneNumber?.countryCode);
    if (phoneNumber?.number?.length !== country?.numberLength) {
      errors.phoneNumber = t("error_messages.invalid_phone_number_length");
    }
    return errors;
  };

  const formik = useFormik({
    initialValues: initialState,
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    validate: (values) => validate(values),
    onSubmit: async (values, { resetForm }) => {
      try {
        if (values.action === PHONE_NUMBER_ACTION.REMOVE)
          values.phoneNumber = null;
        values.token = authenticateToken as string;
        await updatePhoneNumber(values).unwrap();
        resetForm();
        snack.success(
          values.action === PHONE_NUMBER_ACTION.REMOVE
            ? t("user_messages.phone_number_removed_successfully")
            : t("user_messages.phone_number_changed_successfully"),
        );
        handleModalClose();
      } catch (error) {
        showResponseErrors(error);
      }
    },
  });

  const handleModalClose = () => {
    formik.resetForm();
    setView(hasPhoneNumber ? ModalView.SELECTION : ModalView.FORM);
    onClose();
  };

  const handleEscape = () => {
    if (isLoading) return;
    if (view === ModalView.FORM && hasPhoneNumber) {
      handleBackToSelection();
    } else {
      handleModalClose();
    }
  };

  const handleSelectAction = (selectedAction: PHONE_NUMBER_ACTION) => {
    formik.setFieldValue("action", selectedAction);
    if (selectedAction === PHONE_NUMBER_ACTION.UPDATE) setView(ModalView.FORM);
    else if (selectedAction === PHONE_NUMBER_ACTION.REMOVE) formik.submitForm();
  };

  const handleBackToSelection = () => {
    setView(ModalView.SELECTION);
    formik.setFieldValue("action", null);
    formik.setFieldValue("phoneNumber", user?.phoneNumber || null);
    formik.setTouched({});
  };

  const isFormValid = () => {
    const { phoneNumber } = formik.values;
    return (
      phoneNumber?.fullPhoneNumber &&
      phoneNumber?.countryCode &&
      phoneNumber?.number
    );
  };

  const isPhoneNumberChanged = () => {
    const current = user?.phoneNumber;
    const next = formik.values.phoneNumber;
    if (!current || !next) return true;
    return (
      current.fullPhoneNumber !== next.fullPhoneNumber ||
      current.countryCode !== next.countryCode
    );
  };

  useEffect(() => {
    if (!open) return;

    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === "Enter" && view === ModalView.FORM) {
        if (!isLoading && isFormValid() && isPhoneNumberChanged())
          formik.submitForm();
      }
    };

    document.addEventListener("keydown", handleKeyDown);
    return () => document.removeEventListener("keydown", handleKeyDown);
  }, [open, formik, view]);

  return (
    <Modal
      open={open}
      onClose={handleEscape}
      onMouseDown={(e) =>
        e.target === e.currentTarget && !isLoading && handleModalClose()
      }
    >
      <div className="bg-(--auth-main-bg) rounded-2xl p-6 sm:p-8 max-w-[480px] w-full shadow-(--card-shadow) animate-fade-in mx-auto">
        {/* Header */}
        <div className="mb-7">
          <h2 className="text-2xl font-bold text-(--text-primary) mb-2">
            {view === ModalView.SELECTION
              ? t("common.manage_phone_number")
              : t("common.update_phone_number")}
          </h2>
          <p className="text-sm text-(--muted-color) leading-relaxed">
            {view === ModalView.SELECTION
              ? t("common.manage_phone_number_description")
              : t("common.update_phone_number_description")}
          </p>
        </div>

        {view === ModalView.SELECTION ? (
          /* Selection View */
          <div className="flex flex-col gap-4">
            <Button
              onClick={() => handleSelectAction(PHONE_NUMBER_ACTION.UPDATE)}
              disabled={isLoading}
              className="group flex items-center gap-4 p-4 rounded-xl border border-(--input-border) bg-(--input-bg) hover:bg-(--active-bg-2) hover:border-(--primary-color) transition-all duration-200 text-left"
            >
              <div className="w-10 h-10 rounded-lg bg-(--active-bg-2) flex items-center justify-center text-(--primary-color)">
                <Pencil size={20} />
              </div>
              <div className="flex-1">
                <p className="font-semibold text-(--text-primary) leading-none mb-1">
                  {t("common.update_phone_number")}
                </p>
                <p className="text-xs text-(--muted-color)">
                  {t("common.change_your_phone_number")}
                </p>
              </div>
              <ChevronRight size={18} className="text-(--muted-color)" />
            </Button>

            <Button
              onClick={() => handleSelectAction(PHONE_NUMBER_ACTION.REMOVE)}
              disabled={isLoading}
              className="group flex items-center gap-4 p-4 rounded-xl border border-(--input-border) bg-(--input-bg) hover:bg-red-50 dark:hover:bg-red-950/20 hover:border-(--error-color) transition-all duration-200 text-left"
            >
              <div className="w-10 h-10 rounded-lg bg-red-100 dark:bg-red-900/30 flex items-center justify-center text-(--error-color)">
                <Trash size={20} />
              </div>
              <div className="flex-1">
                <p className="font-semibold text-(--text-primary) leading-none mb-1">
                  {t("common.remove_phone_number")}
                </p>
                <p className="text-xs text-(--muted-color)">
                  {t("common.delete_your_phone_number")}
                </p>
              </div>
              <ChevronRight size={18} className="text-(--muted-color) " />
            </Button>

            <Button
              onClick={handleModalClose}
              className="flex-1 px-6 py-3 rounded-xl text-[15px] font-medium bg-(--input-bg) text-(--text-primary) border border-(--input-border) hover:bg-(--input-border) transition-all"
              title={t("common.cancel")}
            />
          </div>
        ) : (
          /* Form View */
          <div className="flex flex-col gap-6">
            <div className="flex flex-col gap-2">
              <PhoneNumberForm
                name="phoneNumber"
                onChange={(value) => formik.setFieldValue("phoneNumber", value)}
                value={formik.values.phoneNumber}
              />
              {formik.touched.phoneNumber && formik.errors.phoneNumber && (
                <span className="text-(--error-color) text-[13px] mt-1 animate-fade-in pl-1">
                  {formik.errors.phoneNumber as string}
                </span>
              )}
            </div>

            <div className="flex gap-3 mt-2">
              <Button
                type="button"
                onClick={
                  hasPhoneNumber ? handleBackToSelection : handleModalClose
                }
                className="flex-1 px-6 py-3 rounded-xl text-[15px] font-medium bg-(--input-bg) text-(--text-primary) border border-(--input-border) hover:bg-(--input-border) transition-all"
                disabled={isLoading}
                title={hasPhoneNumber ? t("common.back") : t("common.cancel")}
              />
              <Button
                type="button"
                onClick={() => formik.handleSubmit()}
                className="flex-1 px-6 py-3 rounded-xl text-[15px] font-semibold bg-(--primary-color) text-white shadow-(--shadow-color) hover:shadow-(--active-shadow) flex items-center justify-center"
                disabled={
                  isLoading || !isFormValid() || !isPhoneNumberChanged()
                }
                isLoading={isLoading}
                title={t("common.save")}
              />
            </div>
          </div>
        )}
      </div>
    </Modal>
  );
};

export default PhoneNumberModal;

```

## File: connectfy-client/src/modules/settings/AccountSettings/router/router.tsx
```typescript
import { RouteObject } from "react-router-dom";
import { lazy } from "react";
import { ROUTER } from "@/common/constants/routet";
import ComponentLoader from "@/components/Loader/Components/ComponentLoader";

const AccountSettings = ComponentLoader(
  lazy(() => import("../ui/AccountSettings")),
);

const routes: RouteObject[] = [
  {
    path: ROUTER.SETTINGS.ACCOUNT,
    element: <AccountSettings />,
  },
];

export default routes;

```

## File: connectfy-client/src/modules/settings/AccountSettings/types/types.ts
```typescript
import {
  DELETE_REASON_CODE,
  PHONE_NUMBER_ACTION,
  TWO_FACTOR_ACTION,
} from "@/common/enums/enums";
import { IPhoneNumber } from "@/modules/auth/types/types";
import { IUser } from "@/modules/profile/types/types";

export type ChangeModalKey =
  | "username"
  | "email"
  | "password"
  | "phone_number"
  | "delete_account"
  | "deactivate_account"
  | "two_factor"
  | null;

export interface IUpdateUsername {
  username: string | null;
  token: string | null;
}

export interface IUpdateEmail {
  email: string | null;
  token: string | null;
}

export interface IUpdatePassword {
  password: string | null;
  confirmPassword: string | null;
  token: string | null;
}

export interface IVerifyChangeEmail {
  token: string | null;
}

export interface IUpdatePhoneNumber {
  token: string | null;
  action: PHONE_NUMBER_ACTION | null;
  phoneNumber: IPhoneNumber | null;
}

export interface IUpdateUsernameResponse extends IUser {}

export interface IUpdateEmailResponse {
  statusCode: number;
}

export interface IUpdatePasswordResponse extends IUser {}

export interface IVerifyChangeEmailResponse extends IUser {}

export interface IUpdatePhoneNumberResponse extends IUser {}

export interface IResetPasswordResponse {
  statusCode: number;
}

export interface ILogoutResponse {
  statusCode: number;
}

export interface IDeleteAccount {
  token: string | null;
  reasonCode: DELETE_REASON_CODE | null;
  reasonDescription: string | null;
}

export interface IDeleteAccountResponse {
  statusCode: number;
}

export interface IDeactivateAccount {
  token: string | null;
}

export interface IDeactivateAccountResponse {
  statusCode: number;
}

export interface IUpdateTwoFactor {
  token: string | null;
  action: TWO_FACTOR_ACTION;
}

export interface IUpdateTwoFactorResponse extends IUser {}

```

## File: connectfy-client/src/modules/settings/AccountSettings/api/api.ts
```typescript
import { baseQuery } from "@/common/api/axiosBaseQuery";
import {
  PHONE_NUMBER_ACTION,
  RESOURCE,
  TWO_FACTOR_ACTION,
} from "@/common/enums/enums";
import { createApi } from "@reduxjs/toolkit/query/react";
import {
  IDeactivateAccount,
  IDeactivateAccountResponse,
  IDeleteAccount,
  ILogoutResponse,
  IUpdateEmail,
  IUpdateEmailResponse,
  IUpdatePassword,
  IUpdatePhoneNumber,
  IUpdateTwoFactor,
  IUpdateUsername,
  IVerifyChangeEmail,
} from "../types/types";
import { API_ENDPOINTS } from "@/common/constants/apiEndpoints";
import { profileApi } from "@/modules/profile/api/api";
import { IUpdateResponse } from "@/common/interfaces/interfaces";

export const accountSettingsApi = createApi({
  reducerPath: RESOURCE.ACCOUNT_SETTINGS,
  baseQuery: baseQuery,
  tagTypes: ["User", "AccountSettings"],
  endpoints: (builder) => ({
    // ====================== UPDATE USERNAME
    updateUsername: builder.mutation<IUpdateResponse, IUpdateUsername>({
      query: (data) => ({
        url: API_ENDPOINTS.USER.CHANGE_USERNAME,
        method: "PATCH",
        body: data,
      }),
      /**
       * Optimistic update: instantly update profileApi.getMe cache.
       * We patch getMe (undefined arg) because getMe takes void in your code.
       * Rollback on failure.
       */
      async onQueryStarted(arg, { dispatch, queryFulfilled }) {
        // apply optimistic change to getMe
        const patchResultForUser = dispatch(
          profileApi.util.updateQueryData("getMe", undefined, (draft: any) => {
            if (!draft) return;

            const format = draft.defaultAvatar?.format || "micah";
            const seed = encodeURIComponent(arg.username!);

            draft.username = arg.username;

            draft.defaultAvatar.seed = arg.username;
            draft.defaultAvatar.url = `https://api.dicebear.com/9.x/${format}/svg?seed=${seed}`;

            if (draft.avatar && !draft.avatar.isCustom && !draft.avatar.key) {
              draft.avatar.url = `https://api.dicebear.com/9.x/${format}/svg?seed=${seed}`;
            }
          }),
        );

        const patchResultForProfile = dispatch(
          profileApi.util.updateQueryData(
            "getAccount",
            undefined,
            (draft: any) => {
              if (!draft) return;

              const format = draft.defaultAvatar?.format || "micah";
              const seed = encodeURIComponent(arg.username!);

              draft.username = arg.username;

              draft.defaultAvatar.seed = arg.username;
              draft.defaultAvatar.url = `https://api.dicebear.com/9.x/${format}/svg?seed=${seed}`;

              if (draft.avatar && !draft.avatar.isCustom && !draft.avatar.key) {
                draft.avatar.url = `https://api.dicebear.com/9.x/${format}/svg?seed=${seed}`;
              }
            },
          ),
        );

        try {
          await queryFulfilled;
        } catch (err) {
          patchResultForUser.undo();
          patchResultForProfile.undo();
        }
      },
      /**
       * Invalidate the User tag so other subscribers (if any) refetch.
       * If the response includes the updated user id, return it; otherwise fallback to LIST.
       */
      invalidatesTags: ["User", "AccountSettings"],
    }),

    // ====================== UPDATE EMAIL
    updateEmail: builder.mutation<IUpdateEmailResponse, IUpdateEmail>({
      query: (data) => ({
        url: API_ENDPOINTS.USER.CHANGE_EMAIL,
        method: "PATCH",
        body: data,
      }),
      invalidatesTags: ["AccountSettings"],
    }),

    // ====================== UPDATE PASSWORD
    updatePassword: builder.mutation<IUpdateResponse, IUpdatePassword>({
      query: (data) => ({
        url: API_ENDPOINTS.USER.CHANGE_PASSWORD,
        method: "PATCH",
        body: data,
      }),
      invalidatesTags: ["AccountSettings"],
    }),

    // ====================== VERIFY CHANGE EMAIL
    verifyChangeEmail: builder.mutation<IUpdateResponse, IVerifyChangeEmail>({
      query: (data) => ({
        url: API_ENDPOINTS.USER.VERIFY_CHANGE_EMAIL,
        method: "PATCH",
        body: data,
      }),

      async onQueryStarted(_, { dispatch, queryFulfilled }) {
        const { data } = await queryFulfilled;

        if (data) {
          dispatch(
            profileApi.util.updateQueryData(
              "getMe",
              undefined,
              (draft: any) => {
                if (draft) {
                  draft.email = data.email;
                }
              },
            ),
          );
        }
      },

      invalidatesTags: ["User", "AccountSettings"],
    }),

    // ====================== UPDATE PHONE NUMBER
    updatePhoneNumber: builder.mutation<IUpdateResponse, IUpdatePhoneNumber>({
      query: (data) => ({
        url: API_ENDPOINTS.USER.CHANGE_PHONE_NUMBER,
        method: "PATCH",
        body: data,
      }),

      async onQueryStarted(arg, { dispatch, queryFulfilled }) {
        const patchResult = dispatch(
          profileApi.util.updateQueryData("getMe", undefined, (draft: any) => {
            if (draft) {
              const action = arg.action;
              draft.phoneNumber =
                action === PHONE_NUMBER_ACTION.UPDATE ? arg.phoneNumber : null;
            }
          }),
        );

        try {
          await queryFulfilled;
        } catch {
          patchResult.undo();
        }
      },
      invalidatesTags: ["User", "AccountSettings"],
    }),

    // ====================== LOGOUT
    logout: builder.mutation<ILogoutResponse, void>({
      query: () => ({
        url: API_ENDPOINTS.AUTH.LOGOUT,
        method: "POST",
      }),
      invalidatesTags: [{ type: "User", id: "LIST" }],
    }),

    // ====================== DELETE ACCOUNT
    deleteAccount: builder.mutation<IUpdateResponse, IDeleteAccount>({
      query: (data) => ({
        url: API_ENDPOINTS.USER.DELETE_ACCOUNT,
        method: "POST",
        body: data,
      }),
      invalidatesTags: [{ type: "User", id: "LIST" }],
    }),

    // ====================== DEACTIVATE ACCOUNT
    deactivateAccount: builder.mutation<
      IDeactivateAccountResponse,
      IDeactivateAccount
    >({
      query: (data) => ({
        url: API_ENDPOINTS.USER.DEACTIVATE_ACCOUNT,
        method: "POST",
        body: data,
      }),
      invalidatesTags: [{ type: "User", id: "LIST" }],
    }),

    updateTwoFactor: builder.mutation<IUpdateResponse, IUpdateTwoFactor>({
      query: (data) => ({
        url: API_ENDPOINTS.USER.UPDATE_TWO_FACTOR,
        method: "PATCH",
        body: data,
      }),

      async onQueryStarted(arg, { dispatch, queryFulfilled }) {
        const patchResult = dispatch(
          profileApi.util.updateQueryData("getMe", undefined, (draft: any) => {
            if (draft) {
              draft.isTwoFactorEnabled =
                arg.action === TWO_FACTOR_ACTION.ENABLE;
            }
          }),
        );

        try {
          await queryFulfilled;
        } catch {
          patchResult.undo();
        }
      },
      invalidatesTags: ["User", "AccountSettings"],
    }),
  }),
});

export const {
  useUpdateUsernameMutation,
  useUpdateEmailMutation,
  useUpdatePasswordMutation,
  useVerifyChangeEmailMutation,
  useUpdatePhoneNumberMutation,
  useLogoutMutation,
  useDeleteAccountMutation,
  useDeactivateAccountMutation,
  useUpdateTwoFactorMutation,
} = accountSettingsApi;

```

## File: connectfy-client/src/modules/notifications/hooks/useSwipe.ts
```typescript
import { useState, useRef } from "react";

export const useSwipe = (
  rightPanelW: number,
  leftPanelW: number,
  // onLeftSwipeComplete?: () => void,
) => {
  const [offset, setOffset] = useState(0);
  const [isRightOpen, setROpen] = useState(false);
  const [isLeftOpen, setLOpen] = useState(false);
  const [trans, setTrans] = useState(false);
  const startX = useRef(0);
  const dragging = useRef(false);

  const snap = (target: number) => {
    setTrans(true);
    setOffset(target);
    setROpen(target < 0);
    setLOpen(target > 0);
    setTimeout(() => setTrans(false), 250);
  };

  const handlers = {
    onPointerDown: (e: React.PointerEvent) => {
      if ((e.target as HTMLElement).closest("button, a, label, input")) return;
      startX.current = e.clientX;
      dragging.current = true;
      setTrans(false);
      (e.currentTarget as HTMLElement).setPointerCapture(e.pointerId);
    },
    onPointerMove: (e: React.PointerEvent) => {
      if (!dragging.current) return;
      const delta = e.clientX - startX.current;
      if (isRightOpen)
        setOffset(Math.max(-rightPanelW, Math.min(0, -rightPanelW + delta)));
      else if (isLeftOpen)
        setOffset(Math.max(0, Math.min(leftPanelW, leftPanelW + delta)));
      else if (delta < 0) setOffset(Math.max(-rightPanelW, delta));
      else if (delta > 0) setOffset(Math.min(leftPanelW, delta));
    },
    onPointerUp: (isSelected: boolean, onToggleSelect: () => void) => {
      if (!dragging.current) return;
      dragging.current = false;
      const THRESHOLD = 40;

      if (offset < 0) {
        const shouldOpen = isRightOpen
          ? offset < -(rightPanelW / 2)
          : offset < -THRESHOLD;
        snap(shouldOpen ? -rightPanelW : 0);
      } else if (offset > 0) {
        const shouldOpen = isLeftOpen
          ? offset > leftPanelW / 2
          : offset > THRESHOLD;
        snap(shouldOpen ? leftPanelW : 0);

        if (shouldOpen && !isSelected) {
          onToggleSelect();
        }
      } else {
        snap(0);
        if (isSelected) onToggleSelect();
      }
    },
  };

  return { offset, trans, snap, handlers };
};

```

## File: connectfy-client/src/modules/notifications/hooks/useNotifications.tsx
```typescript
import toast from "react-hot-toast";
import { useEffect, useRef } from "react";

import { useAuthStore } from "@/store/zustand/useAuthStore";
import { useSockets } from "@/context/SocketContext";
import FriendshipToastContainer from "../ui/components/FriendshipToastContainer";
import {
  LANGUAGE,
  NotificationStatus,
  NotificationType,
} from "@/common/enums/enums";
import { INotification } from "../types/types";
import { notificationSettingsApi } from "@/modules/settings/NotificationSettings/api/api";
import { resolveToastContent } from "@/common/utils/notificationHelpers";
import { notificationApi } from "../api/api";
import { useGeneralSettings } from "@/modules/settings/GeneralSettings/hooks/useGeneralSettings";
import { myFriendsApi } from "@/modules/users/MyFriends/api/api";
import { useDispatch, useStore } from "react-redux";

export const useNotifications = () => {
  const { generalSettings } = useGeneralSettings();
  const { access_token } = useAuthStore();
  const dispatch = useDispatch<any>();
  const store = useStore();
  const { notification: notificationSocket } = useSockets();

  const languageRef = useRef(generalSettings?.language ?? LANGUAGE.EN);

  useEffect(() => {
    languageRef.current = generalSettings?.language ?? LANGUAGE.EN;
  }, [generalSettings?.language]);

  useEffect(() => {
    if (!access_token || !notificationSocket) return;

    const handler = (payload: INotification) => {
      const state = store.getState() as any;
      const queries = state?.notifications?.queries || {};
      Object.values(queries)
        .filter((q: any) => q?.endpointName === "findAll" && q?.data)
        .forEach((q: any) => {
          dispatch(
            notificationApi.util.updateQueryData(
              "findAll",
              q.originalArgs,
              (draft) => {
                const exists = draft.data.some(
                  (item) => item._id === payload._id,
                );
                if (!exists) {
                  draft.data.unshift(payload as any);
                  if (typeof draft.totalCount === "number") {
                    draft.totalCount += 1;
                  }
                }
              },
            ),
          );
        });

      if (
        payload.status === NotificationStatus.Unread ||
        payload.status === NotificationStatus.Pending ||
        payload.status == null
      ) {
        dispatch(
          notificationApi.util.updateQueryData(
            "countUnread",
            undefined,
            (draft) => {
              if (typeof draft !== "number") return 1;
              return draft + 1;
            },
          ),
        );
      }

      if (payload.type === NotificationType.FRIENDSHIP_REQUEST_SENT) {
        dispatch(
          myFriendsApi.util.updateQueryData(
            "requestsCount",
            undefined,
            (draft) => {
              if (typeof draft !== "number") return 1;
              return draft + 1;
            },
          ),
        );
      }

      const settingsResult = notificationSettingsApi.endpoints.getNotificationSettings.select()(state);
      const { showToast, showBody } = resolveToastContent(settingsResult.data);

      if (!showToast) return;

      switch (payload.type) {
        case NotificationType.FRIENDSHIP_REQUEST_SENT:
          if (!payload.resourceId || !payload.actorId) return; // specific check for friendship
          toast.custom(
            (instance) => (
              <FriendshipToastContainer
                toastId={instance.id}
                payload={payload as any}
                visible={instance.visible}
                showBody={showBody}
              />
            ),
            { duration: 5000, position: "top-right" }
          );
          break;
        
        // case NotificationType.NEW_MESSAGE:
        //   toast.custom((instance) => <MessageToastContainer payload={payload} visible={instance.visible} ... />)
        //   break;

        default:
          // Fallback or generic toast logic for system alerts / undefined types
          break;
      }
    };

    notificationSocket.on("notification:new", handler);

    return () => {
      notificationSocket.off("notification:new", handler);
    };
  }, [access_token, notificationSocket, dispatch, store]);
};

```

## File: connectfy-client/src/modules/notifications/constants/notification.configs.ts
```typescript
import { NotificationType } from "@/common/enums/enums";
import {
  Users,
  UserCheck,
  X,
  MessageCircle,
  AlertCircle,
  Bell,
} from "lucide-react";

type IconConfig = {
  Icon: React.ElementType;
  colorClass: string;
  bgClass: string;
};
export const NOTIFICATION_LAYOUT = {
  BTN_W: 72,
  GAP: 6,
  PADDING: 8,
  LEFT_PANEL_W: 52,
  THRESHOLD: 40,
};

export const getPanelWidth = (n: number) =>
  n * NOTIFICATION_LAYOUT.BTN_W +
  (n - 1) * NOTIFICATION_LAYOUT.GAP +
  NOTIFICATION_LAYOUT.PADDING * 2;

export const FRIENDSHIP_TYPES = new Set([
  NotificationType.FRIENDSHIP_REQUEST_SENT,
  NotificationType.FRIENDSHIP_REQUEST_ACCEPTED,
  NotificationType.FRIENDSHIP_REQUEST_DECLINED,
]);

export const ICON_MAP: Partial<Record<NotificationType, IconConfig>> = {
  [NotificationType.FRIENDSHIP_REQUEST_SENT]: {
    Icon: Users,
    colorClass: "text-blue-400",
    bgClass: "bg-blue-400/10",
  },
  [NotificationType.FRIENDSHIP_REQUEST_ACCEPTED]: {
    Icon: UserCheck,
    colorClass: "text-(--primary-color)",
    bgClass: "bg-(--active-bg)",
  },
  [NotificationType.FRIENDSHIP_REQUEST_DECLINED]: {
    Icon: X,
    colorClass: "text-red-400",
    bgClass: "bg-red-400/10",
  },
  [NotificationType.NEW_MESSAGE]: {
    Icon: MessageCircle,
    colorClass: "text-purple-400",
    bgClass: "bg-purple-400/10",
  },
  [NotificationType.SYSTEM_ALERT]: {
    Icon: AlertCircle,
    colorClass: "text-amber-400",
    bgClass: "bg-amber-400/10",
  },
  [NotificationType.ACCOUNT_WARNING]: {
    Icon: AlertCircle,
    colorClass: "text-red-400",
    bgClass: "bg-red-400/10",
  },
};

export const DEFAULT_ICON = {
  Icon: Bell,
  colorClass: "text-(--muted-color)",
  bgClass: "bg-(--active-bg)",
};

```

## File: connectfy-client/src/modules/notifications/utils/notification.utils.ts
```typescript
import { LANGUAGE } from "@/common/enums/enums";

export const getLangText = (
  msg: Record<string, string> | null | undefined,
  lang: string,
): string | null => {
  if (!msg) return null;
  return msg[lang] ?? msg[LANGUAGE.EN] ?? null;
};

```

## File: connectfy-client/src/modules/notifications/ui/Notifications.tsx
```typescript
import { useState, useMemo, useCallback } from "react";
import { useNavigate } from "react-router-dom";
import { useTranslation } from "react-i18next";
import { useDispatch, useStore } from "react-redux";
import {
  ArrowLeft,
  ArrowRight,
  Bell,
  CheckCheck,
  InboxIcon,
  Trash2,
  X,
} from "lucide-react";

import {
  LANGUAGE,
  NotificationStatus,
  NotificationType,
} from "@/common/enums/enums";
import {
  useFindAllQuery,
  useCountUnreadQuery,
  useMarkReadMutation,
  useMarkUnreadMutation,
  useMarkAllReadMutation,
  useRemoveMutation,
  useRemoveAllMutation,
} from "../api/api";
import { useGeneralSettings } from "@/modules/settings/GeneralSettings/hooks/useGeneralSettings";
import {
  useAcceptFriendRequestMutation,
  useDeclineFriendRequestMutation,
} from "@/modules/users/MyFriends/api/api";
import { useErrors } from "@/hooks/useErrors";
import { ROUTER } from "@/common/constants/routet";
import { patchFriendshipNotificationMetadata } from "../api/api";

import NotificationFilters, {
  FilterType,
} from "./components/NotificationFilters";
import NotificationItem from "./components/NotificationItem";
import NotificationSkeleton from "@/components/Skeleton/notification/NotificationSkeleton";
import Button from "@/components/ui/CustomButton/Button/Button";

// ─── actorId populated object ─────────────────────────────────────────────────
interface IActorPopulated {
  _id: string;
  firstName: string;
  lastName: string;
  username: string;
  avatar: string | null;
}

const getActorId = (
  actorId: string | IActorPopulated | null,
): string | null => {
  if (!actorId) return null;
  return typeof actorId === "string" ? actorId : actorId._id;
};

// ─── Constants ────────────────────────────────────────────────────────────────
const PAGE_LIMIT = 20;

const FILTER_PARAMS: Record<
  FilterType,
  { status?: NotificationStatus; type?: NotificationType }
> = {
  all: {},
  unread: { status: NotificationStatus.Unread },
  friendship: { type: NotificationType.FRIENDSHIP_REQUEST_SENT },
  system: { type: NotificationType.SYSTEM_ALERT },
};

// ─── Empty State ──────────────────────────────────────────────────────────────
const EmptyState = ({ label }: { label: string }) => (
  <div className="flex flex-col items-center justify-center py-16 gap-3 text-(--muted-color)">
    <InboxIcon size={36} strokeWidth={1.2} />
    <p className="text-sm">{label}</p>
  </div>
);

// ─── Page ─────────────────────────────────────────────────────────────────────
const Notifications = () => {
  const { t } = useTranslation();
  const navigate = useNavigate();
  const { generalSettings } = useGeneralSettings();
  const language = (generalSettings?.language ?? LANGUAGE.EN) as LANGUAGE;
  const { showResponseErrors } = useErrors();
  const dispatch = useDispatch();
  const store = useStore();

  const [filter, setFilter] = useState<FilterType>("all");
  const [skip, setSkip] = useState(0);
  const [selectedIds, setSelectedIds] = useState<Set<string>>(new Set());
  const [selectedUnreadIds, setSelectedUnreadIds] = useState<Set<string>>(
    new Set(),
  );

  const handleFilterChange = (f: FilterType) => {
    setFilter(f);
    setSkip(0);
    setSelectedIds(new Set());
    setSelectedUnreadIds(new Set());
  };

  const queryParams = useMemo(
    () => ({ skip, limit: PAGE_LIMIT, ...FILTER_PARAMS[filter] }),
    [filter, skip],
  );

  const { data, isLoading, isFetching } = useFindAllQuery(queryParams);
  const { data: unreadCount } = useCountUnreadQuery();
  const [markRead] = useMarkReadMutation();
  const [markUnread] = useMarkUnreadMutation();
  const [markAllRead, { isLoading: isMarkAllLoading }] =
    useMarkAllReadMutation();
  const [remove, { isLoading: isRemoveLoading }] = useRemoveMutation();
  const [removeAll, { isLoading: isRemoveAllLoading }] = useRemoveAllMutation();
  const [acceptFriendRequest, { isLoading: isAcceptLoading }] =
    useAcceptFriendRequestMutation();
  const [declineFriendRequest, { isLoading: isDeclineLoading }] =
    useDeclineFriendRequestMutation();

  const notifications = data?.data ?? [];
  const total = data?.totalCount ?? 0;
  const hasMore = skip + PAGE_LIMIT < total;
  const hasSelection = selectedIds.size > 0;

  // ── Selection helpers ────────────────────────────────────────────────────────
  const toggleSelect = useCallback((id: string) => {
    setSelectedIds((prev) => {
      const next = new Set(prev);
      next.has(id) ? next.delete(id) : next.add(id);
      return next;
    });
  }, []);

  const toggleUnreadSelect = useCallback((id: string) => {
    setSelectedUnreadIds((prev) => {
      const next = new Set(prev);
      next.has(id) ? next.delete(id) : next.add(id);
      return next;
    });
  }, []);

  const clearSelection = () => {
    setSelectedIds(new Set());
    setSelectedUnreadIds(new Set());
  };

  // ── Handlers ─────────────────────────────────────────────────────────────────
  const handleMarkAllRead = async () => {
    try {
      const ids = Array.from(selectedUnreadIds);
      await markAllRead({ _ids: ids.length ? ids : undefined }).unwrap();
      clearSelection();
    } catch (e) {
      showResponseErrors(e);
    }
  };

  const handleMarkRead = async (_id: string) => {
    try {
      await markRead({ _id }).unwrap();
    } catch (e) {
      showResponseErrors(e);
    }
  };

  const handleMarkUnread = async (_id: string) => {
    try {
      await markUnread({ _id }).unwrap();
    } catch (e) {
      showResponseErrors(e);
    }
  };

  const handleDelete = async (_id: string) => {
    try {
      await remove({ _id }).unwrap();
      setSelectedIds((prev) => {
        const n = new Set(prev);
        n.delete(_id);
        return n;
      });
      setSelectedUnreadIds((prev) => {
        const n = new Set(prev);
        n.delete(_id);
        return n;
      });
    } catch (e) {
      showResponseErrors(e);
    }
  };

  const handleDeleteSelected = async () => {
    try {
      await removeAll({ _ids: Array.from(selectedIds) }).unwrap();
      clearSelection();
    } catch (e) {
      showResponseErrors(e);
    }
  };

  const handleSyncError = (error: any) => {
    const additional =
      error?.error?.additional ?? error?.data?.additional ?? error?.additional;
    if (additional?.relationship || additional?.alreadyCancelled) return;
    showResponseErrors(error);
  };

  const handleAcceptFriendRequestInline = async (
    friendshipId: string,
    userId: string,
    notificationId: string,
  ) => {
    try {
      await acceptFriendRequest({ friendshipId, userId }).unwrap();
      patchFriendshipNotificationMetadata(dispatch, store.getState(), userId, {
        isAccepted: true,
      });
      void markRead({ _id: notificationId })
        .unwrap()
        .catch(() => {});
    } catch (error: any) {
      handleSyncError(error);
    }
  };

  const handleDeclineFriendRequestInline = async (
    friendshipId: string,
    userId: string,
    notificationId: string,
  ) => {
    try {
      await declineFriendRequest({ friendshipId, userId }).unwrap();
      patchFriendshipNotificationMetadata(dispatch, store.getState(), userId, {
        isDeclined: true,
      });
      void markRead({ _id: notificationId })
        .unwrap()
        .catch(() => {});
    } catch (error: any) {
      handleSyncError(error);
    }
  };

  return (
    <div className="flex flex-col h-full">
      {/* ── Sticky header ──────────────────────────────────────────────── */}
      <div className="px-4 pt-6 pb-3 bg-(--bg-color)">
        {/* ── Default header ──────────────────────────────────────────── */}
        {!hasSelection ? (
          <div className="flex items-center justify-between mb-4">
            <div className="flex items-center gap-2">
              <Bell size={19} className="text-(--primary-color)" />
              <h1 className="text-base font-semibold text-(--text-color)">
                {t("notification.header")}
              </h1>
              {!!unreadCount && (
                <span
                  className="bg-(--primary-color) text-white text-[10px] font-bold
                  px-1.5 py-0.5 rounded-full leading-none"
                >
                  {unreadCount > 99 ? "99+" : unreadCount}
                </span>
              )}
            </div>

            <Button
              onClick={handleMarkAllRead}
              disabled={isMarkAllLoading || !unreadCount}
              isLoading={isMarkAllLoading}
              className="flex items-center gap-1.5 text-xs font-medium px-3 py-1.5 rounded-lg
                text-(--primary-color) border border-(--primary-color)
                hover:bg-(--active-bg)
                transition-colors duration-600"
              icon={<CheckCheck size={15} />}
              title={t("notification.mark_all_as_read")}
            />
          </div>
        ) : (
          /* ── Selection header ─────────────────────────────────────── */
          <div className="flex items-center justify-between mb-4">
            <div className="flex items-center gap-2">
              <Button
                onClick={clearSelection}
                className="p-1 rounded-lg hover:bg-(--active-bg) text-(--muted-color)
                  hover:text-(--text-color) transition-colors"
                icon={<X size={19} />}
                tooltip={t("common.close")}
              />
              <span className="text-sm font-semibold text-(--text-color)">
                {selectedIds.size} {t("notification.selected")}
              </span>
            </div>

            <div className="flex gap-2">
              <Button
                onClick={handleMarkAllRead}
                disabled={
                  isMarkAllLoading || !unreadCount || !selectedUnreadIds.size
                }
                isLoading={isMarkAllLoading}
                className="flex items-center gap-1.5 text-xs font-medium px-3 py-1.5 rounded-lg
                text-(--primary-color) border border-(--primary-color)
                hover:bg-(--active-bg)
                transition-colors duration-600"
                icon={<CheckCheck size={15} />}
                title={t("notification.mark_all_as_read")}
              />

              <Button
                onClick={handleDeleteSelected}
                disabled={isRemoveAllLoading}
                isLoading={isRemoveAllLoading}
                className="flex items-center gap-1.5 text-xs font-medium px-3 py-1.5 rounded-lg
                bg-(--error-color) text-white hover:opacity-90 border border-(--error-color)
                transition-all duration-600"
                icon={<Trash2 size={15} />}
                title={t("notification.remove_selected")}
              />
            </div>
          </div>
        )}

        <NotificationFilters
          active={filter}
          onChange={handleFilterChange}
          unreadCount={unreadCount ?? 0}
        />
      </div>

      {/* ── Scrollable list ─────────────────────────────────────────────── */}
      <div className="flex-1 overflow-y-auto px-4 pb-6">
        {!isLoading && notifications.length > 0 && !hasSelection && (
          <div className="flex items-center justify-end mb-2 gap-2">
            <div className="flex items-center gap-1">
              <ArrowLeft size={12} className="text-(--muted-color)" />
              <p className="text-[11px] text-(--muted-color) select-none">
                {t("notification.swipe_left_hint")}
              </p>
            </div>

            <div className="w-1.5 h-1.5 bg-(--muted-color) rounded-full"></div>

            <div className="flex items-center gap-1">
              <p className="text-[11px] text-(--muted-color) select-none">
                {t("notification.swipe_right_hint")}
              </p>
              <ArrowRight size={12} className="text-(--muted-color)" />
            </div>
          </div>
        )}

        <div className="space-y-2">
          {isLoading ? (
            Array.from({ length: 6 }).map((_, i) => (
              <NotificationSkeleton key={i} />
            ))
          ) : notifications.length === 0 ? (
            <EmptyState label={t("notification.empty")} />
          ) : (
            notifications.map((notif) => {
              const actorId = getActorId(
                notif.actorId as string | IActorPopulated | null,
              );
              return (
                <NotificationItem
                  key={notif._id}
                  notification={notif}
                  language={language}
                  selectedIds={selectedIds}
                  isSelected={selectedIds.has(notif._id)}
                  onToggleSelect={() => {
                    toggleSelect(notif._id);
                    if (notif.status === NotificationStatus.Unread) {
                      toggleUnreadSelect(notif._id);
                    }
                  }}
                  onMarkRead={() => handleMarkRead(notif._id)}
                  onMarkUnread={() => handleMarkUnread(notif._id)}
                  onDelete={() => handleDelete(notif._id)}
                  onMessage={() => {}}
                  onAccept={
                    notif.resourceId && actorId
                      ? () =>
                          void handleAcceptFriendRequestInline(
                            notif.resourceId!,
                            actorId,
                            notif._id,
                          )
                      : undefined
                  }
                  onDecline={
                    notif.resourceId && actorId
                      ? () =>
                          void handleDeclineFriendRequestInline(
                            notif.resourceId!,
                            actorId,
                            notif._id,
                          )
                      : undefined
                  }
                  onViewProfile={
                    actorId
                      ? () => navigate(`${ROUTER.USERS.PROFILE}/${actorId}`)
                      : undefined
                  }
                  isActionLoading={
                    isAcceptLoading || isDeclineLoading || isRemoveLoading
                  }
                />
              );
            })
          )}

          {hasMore && (
            <Button
              onClick={() => setSkip((s) => s + PAGE_LIMIT)}
              disabled={isFetching}
              isLoading={isFetching}
              className="w-full py-2.5 text-sm text-(--primary-color) font-medium
                hover:bg-(--active-bg-2) rounded-xl transition-colors"
              title={t("common.load_more")}
              loadingText={t("common.loading")}
              hideTitleInMobile={false}
            />
          )}
        </div>
      </div>
    </div>
  );
};

export default Notifications;

```

## File: connectfy-client/src/modules/notifications/ui/components/FriendshipToastContainer.tsx
```typescript
import { useRef, useEffect } from "react";
import { useDispatch, useStore } from "react-redux";
import { useTranslation } from "react-i18next";
import FriendshipToast from "@/components/ui/CustomToast/FriendshipToast";
import { useErrors } from "@/hooks/useErrors";
import { LANGUAGE } from "@/common/enums/enums";
import { useGeneralSettings } from "@/modules/settings/GeneralSettings/hooks/useGeneralSettings";
import {
  patchFriendshipNotificationMetadata,
  useMarkReadMutation,
} from "../../api/api";
import {
  useDeclineFriendRequestMutation,
  useAcceptFriendRequestMutation,
  myFriendsApi,
} from "@/modules/users/MyFriends/api/api";
import { IFriendshipNotificationPayload } from "../../types/types";

interface Props {
  payload: IFriendshipNotificationPayload;
  toastId: string;
  visible: boolean;
  showBody?: boolean;
}

const FriendshipToastContainer = ({ payload, toastId, visible, showBody }: Props) => {
  const { generalSettings } = useGeneralSettings();
  const dispatch = useDispatch<any>();
  const store = useStore();
  const { showResponseErrors } = useErrors();
  const [acceptFriendRequest, { isLoading: isAcceptLoading }] =
    useAcceptFriendRequestMutation();
  const [declineFriendRequest, { isLoading: isDeclineLoading }] =
    useDeclineFriendRequestMutation();
  const [markRead] = useMarkReadMutation();
  const { t } = useTranslation();

  const languageRef = useRef(generalSettings?.language ?? LANGUAGE.EN);

  useEffect(() => {
    languageRef.current = generalSettings?.language ?? LANGUAGE.EN;
  }, [generalSettings?.language]);

  const handleSyncError = (error: any) => {
    const additional =
      error?.error?.additional ??
      error?.data?.additional ??
      error?.additional;
    if (additional?.relationship || additional?.alreadyCancelled) return;
    showResponseErrors(error);
  };

  const handleAccept = async () => {
    const friendshipId = payload.resourceId as string;
    const userId = payload.actorId as string;
    if (!friendshipId || !userId) return;

    try {
      await acceptFriendRequest({ friendshipId, userId }).unwrap();
      patchFriendshipNotificationMetadata(
        dispatch,
        store.getState(),
        userId,
        { isAccepted: true }
      );
      void markRead({ _id: payload._id }).unwrap().catch(() => {});
    } catch (error: any) {
      handleSyncError(error?.error ?? error);
    } finally {
      dispatch(
        myFriendsApi.util.updateQueryData("requestsCount", undefined, (draft) => {
          if (typeof draft !== "number") return 0;
          return draft > 0 ? draft - 1 : 0;
        })
      );
    }
  };

  const handleDecline = async () => {
    const friendshipId = payload.resourceId as string;
    const userId = payload.actorId as string;
    if (!friendshipId || !userId) return;

    try {
      await declineFriendRequest({ friendshipId, userId }).unwrap();
      patchFriendshipNotificationMetadata(
        dispatch,
        store.getState(),
        userId,
        { isDeclined: true }
      );
      void markRead({ _id: payload._id }).unwrap().catch(() => {});
    } catch (error: any) {
      handleSyncError(error?.error ?? error);
    } finally {
      dispatch(
        myFriendsApi.util.updateQueryData("requestsCount", undefined, (draft) => {
          if (typeof draft !== "number") return 0;
          return draft > 0 ? draft - 1 : 0;
        })
      );
    }
  };

  const lang = languageRef.current;

  return (
    <FriendshipToast
      toastId={toastId}
      actorUsername={payload.metadata?.actorUsername as string}
      actorAvatar={(payload.metadata?.actorAvatar as string) ?? null}
      title={payload.title?.[lang] ?? null}
      body={payload.body?.[lang] ?? null}
      isLoading={isAcceptLoading || isDeclineLoading}
      acceptLabel={t("common.accept", { lng: lang })}
      declineLabel={t("common.decline", { lng: lang })}
      onAccept={handleAccept}
      onDecline={handleDecline}
      type={payload.type}
      visible={visible}
      showBody={showBody}
    />
  );
};

export default FriendshipToastContainer;

```

## File: connectfy-client/src/modules/notifications/ui/components/NotificationItem.tsx
```typescript
import { useEffect } from "react";
import { useTranslation } from "react-i18next";
import {
  Check,
  X,
  UserCheck,
  Eye,
  RotateCcw,
  Trash2,
  MessageCircle,
} from "lucide-react";
import { INotification } from "../../types/types";
import {
  NotificationStatus,
  NotificationType,
  LANGUAGE,
} from "@/common/enums/enums";
import Button from "@/components/ui/CustomButton/Button/Button";
import Checkbox from "@/components/ui/CustomCheckbox/Checkbox/Checkbox";

// Externalized Modules
import { getLangText } from "../../utils/notification.utils";
import { useSwipe } from "../../hooks/useSwipe";
import {
  NOTIFICATION_LAYOUT,
  getPanelWidth,
  ICON_MAP,
  DEFAULT_ICON,
  FRIENDSHIP_TYPES,
} from "../../constants/notification.configs";
import { getRelativeTime } from "@/common/utils/formatValues";

interface Props {
  notification: INotification;
  language: LANGUAGE;
  isSelected: boolean;
  onToggleSelect: () => void;
  onMarkRead: () => void;
  onMarkUnread?: () => void;
  onDelete: () => void;
  onAccept?: () => void;
  onDecline?: () => void;
  onViewProfile?: () => void;
  onMessage?: () => void;
  isActionLoading?: boolean;
  selectedIds: Set<string>;
}

const NotificationItem = ({
  notification,
  language,
  isSelected,
  onToggleSelect,
  onMarkRead,
  onMarkUnread,
  onDelete,
  onAccept,
  onDecline,
  onViewProfile,
  onMessage,
  isActionLoading,
  selectedIds,
}: Props) => {
  const { t } = useTranslation();

  const isRead = notification.status === NotificationStatus.Read;
  const isAccepted = notification.status === NotificationStatus.Accepted || !!notification.metadata?.isAccepted;
  const isDeclined = notification.status === NotificationStatus.Declined || !!notification.metadata?.isDeclined;
  const isPending = notification.status === NotificationStatus.Pending || (!isAccepted && !isDeclined);
  const isFriendship = FRIENDSHIP_TYPES.has(notification.type);

  const showFriendActions =
    notification.type === NotificationType.FRIENDSHIP_REQUEST_SENT &&
    isPending &&
    !!onAccept &&
    !!onDecline;

  const showViewProfile = isFriendship && !!onViewProfile;
  const { Icon, colorClass, bgClass } =
    ICON_MAP[notification.type] ?? DEFAULT_ICON;

  const rightPanelW = getPanelWidth(1 + (showViewProfile ? 1 : 0) + 1);
  const { offset, trans, snap, handlers } = useSwipe(
    rightPanelW,
    NOTIFICATION_LAYOUT.LEFT_PANEL_W,
  );

  const actionBtnBaseClass = `flex flex-col items-center justify-center gap-1 px-0.5 py-2.5 rounded-xl
    text-white text-[10px] font-semibold cursor-pointer
    transition-all duration-150 active:scale-95 leading-tight text-center`;

  useEffect(() => {
    if (selectedIds.size === 0) snap(0);
  }, [selectedIds]);

  return (
    <div className="relative rounded-xl overflow-hidden">
      {/* ── Left panel: Checkbox ── */}
      <div
        className={`absolute inset-y-0 left-0 flex items-center justify-center bg-(--active-bg-2)
          transition-opacity duration-200 ${offset > 0 ? "opacity-100" : "opacity-0 pointer-events-none"}`}
        style={{ width: NOTIFICATION_LAYOUT.LEFT_PANEL_W }}
      >
        <Checkbox
          checked={isSelected}
          onChange={() => {
            onToggleSelect();
            if (isSelected) snap(0);
          }}
        />
      </div>

      {/* ── Right panel: Actions ── */}
      <div
        className={`absolute inset-y-0 right-0 flex items-center justify-center gap-1.5 px-2
          transition-opacity duration-200 ${offset < 0 ? "opacity-100" : "opacity-0 pointer-events-none"}`}
        style={{ width: rightPanelW }}
      >
        <Button
          onClick={() => {
            isRead ? onMarkUnread?.() : onMarkRead();
            snap(0);
          }}
          style={{ width: NOTIFICATION_LAYOUT.BTN_W }}
          className={`${actionBtnBaseClass} ${isRead ? "bg-indigo-500" : "bg-(--primary-color)"}`}
        >
          {isRead ? <RotateCcw size={14} /> : <Check size={14} />}
          <span className="w-full px-1 overflow-hidden whitespace-nowrap text-ellipsis">
            {isRead ? t("notification.unread") : t("notification.read")}
          </span>
        </Button>

        {showViewProfile && (
          <Button
            onClick={() => {
              onViewProfile();
              snap(0);
            }}
            style={{ width: NOTIFICATION_LAYOUT.BTN_W }}
            className={`${actionBtnBaseClass} bg-(--card-bg) border border-(--input-border) text-(--primary-color)`}
          >
            <Eye size={14} className="text-(--text-color)" />
            <span className="w-full px-1 overflow-hidden whitespace-nowrap text-ellipsis text-(--text-color)">
              {t("notification.profile")}
            </span>
          </Button>
        )}

        <Button
          onClick={() => {
            onDelete();
            snap(0);
          }}
          style={{ width: NOTIFICATION_LAYOUT.BTN_W }}
          className={`${actionBtnBaseClass} bg-(--error-color)`}
        >
          <Trash2 size={14} />
          <span className="w-full px-1 overflow-hidden whitespace-nowrap text-ellipsis">
            {t("common.remove")}
          </span>
        </Button>
      </div>

      {/* ── Main card ── */}
      <div
        className={`relative border rounded-xl cursor-grab active:cursor-grabbing select-none touch-pan-y transition-colors duration-200
          ${isSelected ? "bg-(--active-bg-2) border-(--primary-color)/40" : "bg-(--card-bg) border-(--input-border)"}`}
        style={{
          transform: `translateX(${offset}px)`,
          transition: trans
            ? "transform 0.25s cubic-bezier(0.25, 0.46, 0.45, 0.94)"
            : "none",
          willChange: "transform",
        }}
        {...handlers}
        onPointerUp={() => handlers.onPointerUp(isSelected, onToggleSelect)}
      >
        {!isRead && (
          <span className="absolute left-0 inset-y-0 w-[3px] rounded-l-xl bg-(--primary-color)" />
        )}

        <div className="flex items-center gap-3 px-4 py-2.5 relative">
          <div
            className={`shrink-0 w-8 h-8 rounded-full overflow-hidden ${bgClass} flex items-center justify-center`}
          >
            {(notification.actorId as any)?.avatar ? (
              <img
                src={(notification.actorId as any).avatar}
                alt="avatar"
                className="w-full h-full object-cover"
              />
            ) : (
              <Icon size={15} className={colorClass} />
            )}
          </div>

          <div className="flex-1 min-w-0">
            <div className="flex items-center justify-between gap-2">
              <p
                className={`text-sm leading-tight line-clamp-1 ${!isRead ? "font-semibold text-(--text-color)" : "font-medium text-(--muted-color)"}`}
              >
                {getLangText(notification.title as any, language)}
              </p>
              <span className="shrink-0 text-[11px] text-(--muted-color)">
                {getRelativeTime(notification.createdAt)}
              </span>
            </div>

            {notification.body && (
              <p className="text-xs text-(--muted-color) mt-0.5 leading-snug line-clamp-1">
                {getLangText(notification.body as any, language)}
              </p>
            )}

            {showFriendActions && (
              <div className="flex items-center gap-1.5 mt-2">
                <Button
                  onClick={onAccept}
                  disabled={isActionLoading}
                  className="flex items-center gap-1 text-xs font-medium px-2.5 py-1 rounded-lg bg-(--primary-color) text-white"
                >
                  <UserCheck size={12} /> {t("common.accept")}
                </Button>
                <Button
                  onClick={onDecline}
                  disabled={isActionLoading}
                  className="flex items-center gap-1 text-xs font-medium px-2.5 py-1 rounded-lg border border-(--input-border) text-(--muted-color)"
                >
                  <X size={12} /> {t("common.decline")}
                </Button>
              </div>
            )}
            {notification.type ===
              NotificationType.FRIENDSHIP_REQUEST_ACCEPTED ||
              (isAccepted && (
                <div className="mt-2">
                  <Button
                    onClick={onMessage}
                    className="flex items-center gap-1 text-xs font-medium px-2.5 py-1 rounded-lg bg-(--primary-color) text-white"
                  >
                    <MessageCircle size={12} /> {t("common.message")}
                  </Button>
                </div>
              ))}
          </div>
        </div>
      </div>
    </div>
  );
};

export default NotificationItem;

```

## File: connectfy-client/src/modules/notifications/ui/components/NotificationFilters.tsx
```typescript
import Button from "@/components/ui/CustomButton/Button/Button";
import { useTranslation } from "react-i18next";

export type FilterType = "all" | "unread" | "friendship" | "system";

interface Props {
  active: FilterType;
  onChange: (f: FilterType) => void;
  unreadCount: number;
  disabled?: boolean;
}

const FILTERS: { key: FilterType; i18nKey: string }[] = [
  { key: "all", i18nKey: "notification.filter.all" },
  { key: "unread", i18nKey: "notification.filter.unread" },
  { key: "friendship", i18nKey: "notification.filter.friendship" },
  { key: "system", i18nKey: "notification.filter.system" },
];

const NotificationFilters = ({
  active,
  onChange,
  unreadCount,
  disabled,
}: Props) => {
  const { t } = useTranslation();

  return (
    <div className="flex gap-2 overflow-x-auto pb-1 scrollbar-hide">
      {FILTERS.map(({ key, i18nKey }) => {
        const isActive = active === key;
        return (
          <Button
            key={key}
            onClick={() => onChange(key)}
            className={`
              relative shrink-0 flex items-center gap-1.5
              px-4 py-1.5 rounded-full text-sm font-medium
              transition-all duration-200 cursor-pointer
              ${
                isActive
                  ? "bg-(--primary-color) text-white shadow-sm"
                  : "bg-(--active-bg-2) text-(--muted-color) hover:bg-(--active-bg) hover:text-(--text-color)"
              }
            `}
            disabled={disabled}
          >
            {t(i18nKey)}
            {key === "unread" && unreadCount > 0 && (
              <span
                className={`
                  text-[10px] font-bold px-1.5 py-0.5 rounded-full leading-none
                  ${isActive ? "bg-white/25 text-white" : "bg-(--primary-color) text-white"}
                `}
              >
                {unreadCount > 99 ? "99+" : unreadCount}
              </span>
            )}
          </Button>
        );
      })}
    </div>
  );
};

export default NotificationFilters;

```

## File: connectfy-client/src/modules/notifications/router/router.tsx
```typescript
import { RouteObject } from "react-router-dom";
import { lazy } from "react";
import { ROUTER } from "@/common/constants/routet";
import ComponentLoader from "@/components/Loader/Components/ComponentLoader";
import NotificationSkeleton from "@/components/Skeleton/notification/NotificationSkeleton";
import { Bell, CheckCheck } from "lucide-react";
import { t } from "i18next";
import NotificationFilters from "../ui/components/NotificationFilters";
import Button from "@/components/ui/CustomButton/Button/Button";

const ProfilePageSkeleton = () => (
  <div className="px-4 py-6">
    <div className="flex items-center justify-between mb-6">
      <div className="flex items-center gap-2">
        <Bell size={20} className="text-(--primary-color)" />
        <h1 className="text-lg font-semibold text-(--text-color)">
          {t("notification.header")}
        </h1>
        <span
          className="bg-(--primary-color) text-white text-[11px] font-bold
              px-2 py-0.5 rounded-full leading-none"
        >
          ...
        </span>
      </div>

      <Button
        className="flex items-center gap-1.5 text-xs font-medium px-3 py-1.5 rounded-lg
            text-(--primary-color) border border-(--primary-color)
            hover:bg-(--active-bg)
            disabled:opacity-40 disabled:cursor-not-allowed
            transition-colors duration-150 cursor-pointer"
        disabled
      >
        <CheckCheck size={14} />
        {t("notification.markAllRead")}
      </Button>
    </div>

    <NotificationFilters
      active="all"
      onChange={() => {}}
      unreadCount={1}
      disabled
    />
    <div className="mt-2 space-y-2">
      {Array.from({ length: 5 }).map((_, i) => (
        <NotificationSkeleton key={i} />
      ))}
    </div>
  </div>
);

const NotificationsPage = ComponentLoader(
  lazy(() => import("../ui/Notifications")),
  <ProfilePageSkeleton />,
);

const routes: RouteObject[] = [
  {
    path: ROUTER.NOTIFICATIONS.MAIN,
    element: <NotificationsPage />,
  },
];

export default routes;

```

## File: connectfy-client/src/modules/notifications/types/types.ts
```typescript
import {
  LANGUAGE,
  NotificationChannel,
  NotificationStatus,
  NotificationType,
} from "@/common/enums/enums";

export interface INotificationMessage {
  [LANGUAGE.EN]: string;
  [LANGUAGE.AZ]: string;
  [LANGUAGE.RU]: string;
  [LANGUAGE.TR]: string;
}

export interface IFriendshipNotificationPayload {
  _id: string;
  recipientId: string;
  actorId: string | null;
  type: NotificationType;
  title: INotificationMessage | null;
  body: INotificationMessage | null;
  status: NotificationStatus;
  resourceId: string | null;
  resourceType: string | null;
  metadata: Record<string, unknown> | null;
  readAt: Date | null;
  expiresAt: Date | null;
  createdAt: Date;
  updatedAt: Date;
}

export interface INotification {
  _id: string;
  recipientId: string;
  actorId: Record<string, any> | string | null;
  type: NotificationType;
  title: INotificationMessage | null;
  body: INotificationMessage | null;
  status: NotificationStatus;
  channel: NotificationChannel;
  resourceId: string | null;
  resourceType: string | null;
  metadata: Record<string, unknown> | null;
  readAt: Date | null;
  expiresAt: Date | null;
  createdAt: Date;
  updatedAt: Date;
}

export interface IFindNotifications {
  skip: number;
  limit: number;
  status?: NotificationStatus;
  type?: NotificationType;
}

export interface IMarkAllRead {
  _ids?: string[];
}

```

## File: connectfy-client/src/modules/notifications/api/api.ts
```typescript
import { baseQuery } from "@/common/api/axiosBaseQuery";
import {
  NotificationStatus,
  NotificationType,
  RESOURCE,
} from "@/common/enums/enums";
import { createApi } from "@reduxjs/toolkit/query/react";
import {
  IFindNotifications,
  INotification,
  IMarkAllRead,
} from "../types/types";
import { API_ENDPOINTS } from "@/common/constants/apiEndpoints";
import {
  IFindAllResponse,
  IRemoveAllData,
  IRemoveData,
  IUpdateResponse,
} from "@/common/interfaces/interfaces";

export const patchFriendshipNotificationMetadata = (
  dispatch: any,
  state: any,
  userId: string,
  metadataPatch: { isAccepted?: boolean; isDeclined?: boolean },
) => {
  const queries = state[RESOURCE.NOTIFICATIONS]?.queries || {};

  Object.values(queries)
    .filter((q: any) => q?.endpointName === "findAll" && q?.data)
    .forEach((q: any) => {
      dispatch(
        notificationApi.util.updateQueryData(
          "findAll",
          q.originalArgs,
          (draft) => {
            const notification = draft.data.find((item: INotification) => {
              const actorId =
                typeof item.actorId === "string" ? null : item.actorId?._id;

              return (
                item.type === NotificationType.FRIENDSHIP_REQUEST_SENT &&
                actorId === userId
              );
            });

            if (notification) {
              notification.metadata = {
                ...(notification.metadata ?? {}),
                ...metadataPatch,
              };
            }
          },
        ),
      );
    });
};

export const notificationApi = createApi({
  reducerPath: RESOURCE.NOTIFICATIONS,
  baseQuery: baseQuery,
  tagTypes: ["Notification", "UnreadCount"],
  endpoints: (builder) => ({
    findAll: builder.query<IFindAllResponse<INotification>, IFindNotifications>(
      {
        query: (data) => ({
          url: API_ENDPOINTS.NOTIFICATION_ACTION_HISTORY.NOTIFICATION.GET_ALL,
          method: "POST",
          body: data,
        }),
        providesTags: ["Notification"],
      },
    ),
    countUnread: builder.query<number, void>({
      query: () => ({
        url: API_ENDPOINTS.NOTIFICATION_ACTION_HISTORY.NOTIFICATION
          .COUNT_UNREAD,
        method: "POST",
      }),
      providesTags: ["UnreadCount"],
    }),
    markRead: builder.mutation<IUpdateResponse, { _id: string }>({
      query: (data) => ({
        url: API_ENDPOINTS.NOTIFICATION_ACTION_HISTORY.NOTIFICATION.MARK_READ,
        method: "POST",
        body: data,
      }),
      async onQueryStarted(args, { dispatch, queryFulfilled, getState }) {
        const { _id } = args;
        const state = getState() as any;
        const queries = state[RESOURCE.NOTIFICATIONS].queries;

        const patches = Object.values(queries)
          .filter((q: any) => q?.endpointName === "findAll" && q?.data)
          .map((q: any) =>
            dispatch(
              notificationApi.util.updateQueryData(
                "findAll",
                q.originalArgs,
                (draft) => {
                  const index = draft.data.findIndex(
                    (item: INotification) => item._id === _id,
                  );
                  if (index !== -1) {
                    draft.data[index].status = NotificationStatus.Read;
                  }
                },
              ),
            ),
          );

        const countUnreadPatch = dispatch(
          notificationApi.util.updateQueryData(
            "countUnread",
            undefined,
            (draft) => {
              return draft - 1;
            },
          ),
        );

        try {
          await queryFulfilled;
        } catch {
          patches.forEach((p) => p.undo());
          countUnreadPatch.undo();
        }
      },
    }),
    markUnread: builder.mutation<IUpdateResponse, { _id: string }>({
      query: (data) => ({
        url: API_ENDPOINTS.NOTIFICATION_ACTION_HISTORY.NOTIFICATION.MARK_UNREAD,
        method: "POST",
        body: data,
      }),
      async onQueryStarted(args, { dispatch, queryFulfilled, getState }) {
        const { _id } = args;
        const state = getState() as any;
        const queries = state[RESOURCE.NOTIFICATIONS].queries;

        const patches = Object.values(queries)
          .filter((q: any) => q?.endpointName === "findAll" && q?.data)
          .map((q: any) =>
            dispatch(
              notificationApi.util.updateQueryData(
                "findAll",
                q.originalArgs,
                (draft) => {
                  const index = draft.data.findIndex(
                    (item: INotification) => item._id === _id,
                  );
                  if (index !== -1) {
                    draft.data[index].status = NotificationStatus.Unread;
                  }
                },
              ),
            ),
          );

        const countUnreadPatch = dispatch(
          notificationApi.util.updateQueryData(
            "countUnread",
            undefined,
            (draft) => {
              if (!draft) return 1;
              return draft + 1;
            },
          ),
        );

        try {
          await queryFulfilled;
        } catch {
          patches.forEach((p) => p.undo());
          countUnreadPatch.undo();
        }
      },
    }),
    markAllRead: builder.mutation<void, IMarkAllRead>({
      query: (data) => ({
        url: API_ENDPOINTS.NOTIFICATION_ACTION_HISTORY.NOTIFICATION
          .MARK_ALL_READ,
        method: "POST",
        body: data,
      }),
      async onQueryStarted(args, { dispatch, queryFulfilled, getState }) {
        const { _ids } = args;
        const state = getState() as any;
        const queries = state[RESOURCE.NOTIFICATIONS].queries;

        const patches = Object.values(queries)
          .filter((q: any) => q?.endpointName === "findAll" && q?.data)
          .map((q: any) =>
            dispatch(
              notificationApi.util.updateQueryData(
                "findAll",
                q.originalArgs,
                (draft) => {
                  if (_ids?.length) {
                    draft.data.forEach((item: INotification) => {
                      if (_ids.includes(item._id)) {
                        item.status = NotificationStatus.Read;
                      }
                    });
                  } else {
                    draft.data.forEach((item: INotification) => {
                      item.status = NotificationStatus.Read;
                    });
                  }
                },
              ),
            ),
          );

        const countUnreadPatch = dispatch(
          notificationApi.util.updateQueryData(
            "countUnread",
            undefined,
            (draft) => {
              if (_ids && _ids.length) {
                draft -= _ids.length;
              } else {
                draft = 0;
              }
              return draft;
            },
          ),
        );

        try {
          await queryFulfilled;
        } catch {
          patches.forEach((p) => p.undo());
          countUnreadPatch.undo();
        }
      },
    }),
    remove: builder.mutation<{ success: boolean }, IRemoveData>({
      query: (data) => ({
        url: API_ENDPOINTS.NOTIFICATION_ACTION_HISTORY.NOTIFICATION.REMOVE,
        method: "POST",
        body: data,
      }),
      async onQueryStarted(args, { dispatch, queryFulfilled, getState }) {
        const { _id } = args;
        const state = getState() as any;
        const queries = state[RESOURCE.NOTIFICATIONS].queries;

        const patches = Object.values(queries)
          .filter((q: any) => q?.endpointName === "findAll" && q?.data)
          .map((q: any) =>
            dispatch(
              notificationApi.util.updateQueryData(
                "findAll",
                q.originalArgs,
                (draft) => {
                  const index = draft.data.findIndex(
                    (item: INotification) => item._id === _id,
                  );
                  if (index !== -1) {
                    draft.data.splice(index, 1);
                    draft?.totalCount && (draft.totalCount -= 1);
                  }
                },
              ),
            ),
          );

        const countUnreadPatch = dispatch(
          notificationApi.util.updateQueryData(
            "countUnread",
            undefined,
            (draft) => {
              return draft - 1;
            },
          ),
        );

        try {
          await queryFulfilled;
        } catch {
          patches.forEach((p) => p.undo());
          countUnreadPatch.undo();
        }
      },
    }),
    removeAll: builder.mutation<{ deletedCount: number }, IRemoveAllData>({
      query: (data) => ({
        url: API_ENDPOINTS.NOTIFICATION_ACTION_HISTORY.NOTIFICATION.REMOVE_ALL,
        method: "POST",
        body: data,
      }),
      async onQueryStarted({ _ids }, { dispatch, queryFulfilled, getState }) {
        const state = getState() as any;
        const queries = state[RESOURCE.NOTIFICATIONS].queries;

        const patches = Object.values(queries)
          .filter((q: any) => q?.endpointName === "findAll" && q?.data)
          .map((q: any) =>
            dispatch(
              notificationApi.util.updateQueryData(
                "findAll",
                q.originalArgs,
                (draft) => {
                  draft.data = draft.data.filter(
                    (item: INotification) => !_ids.includes(item._id),
                  );
                  draft?.totalCount && (draft.totalCount -= _ids.length);
                },
              ),
            ),
          );

        const countUnreadPatch = dispatch(
          notificationApi.util.updateQueryData(
            "countUnread",
            undefined,
            (draft) => {
              return draft - _ids.length;
            },
          ),
        );
        try {
          await queryFulfilled;
        } catch {
          patches.forEach((p) => p.undo());
          countUnreadPatch.undo();
        }
      },
    }),
  }),
});

export const {
  useFindAllQuery,
  useCountUnreadQuery,
  useMarkReadMutation,
  useMarkUnreadMutation,
  useMarkAllReadMutation,
  useRemoveMutation,
  useRemoveAllMutation,
} = notificationApi;

```

## File: connectfy-client/src/modules/auth/constants/validation.ts
```typescript
import {
  IForgotPasswordForm,
  IGoogleSignupForm,
  ILoginForm,
  IResetPasswordForm,
  ISignupForm,
  ISignupVerifyForm,
  ILoginVerifyForm,
} from "../types/types";
import {
  FORGOT_PASSWORD_IDENTIFIER_TYPE,
  GENDER,
  IDENTIFIER_TYPE,
} from "@/common/enums/enums";
import { checkEmptyString } from "@/common/utils/checkValues";
import { TFunction } from "i18next";

// =======================> LOGIN
export const validateLogin = (
  values: ILoginForm,
  t: TFunction,
): Record<string, any> => {
  const { identifier, identifierType, password } = values;
  const errors: Record<string, any> = {};

  if (
    !identifierType ||
    !Object.values(IDENTIFIER_TYPE).includes(identifierType)
  ) {
    errors.identifierType = t("error_messages.identifier_type_is_required");
  }

  if (!identifier || !checkEmptyString(identifier)) {
    errors.identifier = t(
      `error_messages.${identifierType.toLowerCase()}_is_required`,
    );
  }

  if (!password || !checkEmptyString(password)) {
    errors.password = t("error_messages.password_is_required");
  }

  return errors;
};

// =======================> VERIFY LOGIN
export const validateVerifyLogin = (
  values: ILoginVerifyForm,
  t: TFunction,
): Record<string, any> => {
  const { code } = values;
  const errors: Record<string, any> = {};

  if (!code || !checkEmptyString(code)) {
    errors.code = t("error_messages.code_is_required");
  }

  if (code?.length !== 6) {
    errors.code = t("error_messages.invalid_code");
  }

  return errors;
};

// =======================> SIGNUP
export const validateSignup = (
  values: ISignupForm,
  t: TFunction,
): Record<string, any> => {
  const {
    firstName,
    lastName,
    username,
    email,
    gender,
    birthdayDate,
    // phoneNumber,
    password,
    confirm,
  } = values;
  const errors: Record<string, any> = {};

  // const { countryCode, number, fullPhoneNumber } = phoneNumber;

  const usernameRegex = /^[A-Za-z0-9._-]+$/;
  const passwordComplexityRegex =
    /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^A-Za-z0-9]).+$/;

  if (!firstName || !checkEmptyString(firstName))
    errors.firstName = t("error_messages.first_name_is_required");

  if (!lastName || !checkEmptyString(lastName))
    errors.lastName = t("error_messages.last_name_is_required");

  if (!username || !checkEmptyString(username))
    errors.username = t("error_messages.username_is_required");
  else if (!usernameRegex.test(username))
    errors.username = t("error_messages.invalid_username");
  else if (username.length < 3)
    errors.username = t("error_messages.min_length", {
      field: t("common.username"),
      length: 3,
    });
  else if (username.length > 30)
    errors.username = t("error_message.max_length", {
      field: t("common.username"),
      length: 30,
    });

  if (!email || !checkEmptyString(email))
    errors.email = t("error_messages.email_name_is_required");
  else if (!email.includes("@"))
    errors.email = t("error_messages.invalid_email");

  if (!gender || !Object.values(GENDER).includes(gender))
    errors.gender = t("error_messages.gender_is_required");

  if (!birthdayDate)
    errors.birthdayDate = t("error_messages.birthday_is_required");

  // if (
  //   !countryCode ||
  //   !number ||
  //   !fullPhoneNumber ||
  //   !checkEmptyString(countryCode) ||
  //   !checkEmptyString(number) ||
  //   !checkEmptyString(fullPhoneNumber) ||
  //   fullPhoneNumber !== countryCode + number
  // )
  //   errors.phoneNumber = t("error_messages.phone_number_is_required");

  // const currentCountry = COUNTRIES.find((c) => c.code === countryCode);

  // if (currentCountry && number?.length !== currentCountry.numberLength)
  //   errors.phoneNumber = t("error_messages.phone_number_length", {
  //     length: currentCountry.numberLength,
  //   });

  if (!password || !checkEmptyString(password))
    errors.password = t("error_messages.password_is_required");
  else if (
    password.length < 8 ||
    password.length > 30 ||
    !passwordComplexityRegex.test(password)
  )
    errors.password = t("error_messages.password_rule");

  if (confirm !== password)
    errors.confirm = t("error_messages.password_mismatch");

  return errors;
};

// =======================> VERIFY SIGNUP
export const validateVerifySignup = (
  values: ISignupVerifyForm,
  t: TFunction,
): Record<string, any> => {
  const { verifyCode } = values;
  const errors: Record<string, any> = {};

  if (!verifyCode || !checkEmptyString(verifyCode))
    errors.verifyCode = t("error_messages.code_is_required");

  if (verifyCode?.length !== 6)
    errors.verifyCode = t("error_messages.invalid_code");

  return errors;
};

// =======================> FORGOT PASSWORD
export const validateForgotPassword = (
  values: IForgotPasswordForm,
  t: TFunction,
): Record<string, any> => {
  const { identifierType, identifier } = values;
  const errors: Record<string, any> = {};

  if (
    !identifierType ||
    !Object.keys(FORGOT_PASSWORD_IDENTIFIER_TYPE).includes(identifierType)
  )
    errors.identifierType = t("error_messages.identifier_type_is_required");

  if (!identifier || !checkEmptyString(identifier))
    errors.identifier = t(
      `error_messages.${identifierType.toLowerCase()}_is_required`,
    );

  return errors;
};

// =======================> RESET PASSWORD
export const validateResetPassword = (
  values: IResetPasswordForm,
  t: TFunction,
): Record<string, any> => {
  const { password, confirmPassword } = values;
  const errors: Record<string, any> = {};

  const passwordComplexityRegex =
    /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^A-Za-z0-9]).+$/;

  if (!password || !checkEmptyString(password))
    errors.password = t("error_messages.password_is_required");
  else if (
    password.length < 8 ||
    password.length > 30 ||
    !passwordComplexityRegex.test(password)
  )
    errors.password = t("error_messages.password_rule");

  if (confirmPassword !== password)
    errors.confirmPassword = t("error_messages.password_mismatch");

  return errors;
};

// =======================> GOOGLE SIGNUP
export const valdiateGoogleSignup = (
  values: IGoogleSignupForm,
  t: TFunction,
): Record<string, any> => {
  const { username, gender, birthdayDate } = values;
  const errors: Record<string, any> = {};

  // const { countryCode, number, fullPhoneNumber } = phoneNumber;

  const usernameRegex = /^[A-Za-z0-9._-]+$/;

  if (!username || !checkEmptyString(username))
    errors.username = t("error_messages.username_is_required");
  else if (!usernameRegex.test(username))
    errors.username = t("error_messages.invalid_username");
  else if (username.length < 3)
    errors.username = t("error_messages.min_length", { length: 3 });
  else if (username.length > 30)
    errors.username = t("error_message.max_length", { length: 30 });

  if (!gender || !Object.values(GENDER).includes(gender))
    errors.gender = t("error_messages.gender_is_required");

  if (!birthdayDate)
    errors.birthdayDate = t("error_messages.birthday_is_required");

  // if (
  //   !countryCode ||
  //   !number ||
  //   !fullPhoneNumber ||
  //   !checkEmptyString(countryCode) ||
  //   !checkEmptyString(number) ||
  //   !checkEmptyString(fullPhoneNumber) ||
  //   fullPhoneNumber !== countryCode + number
  // )
  //   errors.phoneNumber = t("error_messages.phone_number_is_required");

  // const currentCountry = COUNTRIES.find((c) => c.code === countryCode);

  // if (currentCountry && number?.length !== currentCountry.numberLength)
  //   errors.phoneNumber = t("error_messages.phone_number_length", {
  //     length: currentCountry.numberLength,
  //   });

  return errors;
};

```

## File: connectfy-client/src/modules/auth/constants/intialState.ts
```typescript
import {
  IForgotPasswordForm,
  IGoogleSignupForm,
  ILoginForm,
  ILoginVerifyForm,
  IResetPasswordForm,
  ISignupForm,
  ISignupVerifyForm,
} from "../types/types";
import {
  FORGOT_PASSWORD_IDENTIFIER_TYPE,
  IDENTIFIER_TYPE,
  LOCAL_STORAGE_KEYS,
  THEME,
} from "@/common/enums/enums";

const theme = localStorage.getItem(LOCAL_STORAGE_KEYS.APP_THEME) as THEME;

export const loginInitialState: ILoginForm = {
  identifierType: IDENTIFIER_TYPE.USERNAME,
  identifier: null,
  password: null,
};

export const loginVerifyInitialState: ILoginVerifyForm = {
  code: null,
};

export const signupInitialState: ISignupForm = {
  firstName: null,
  lastName: null,
  username: null,
  email: null,
  gender: null,
  password: null,
  confirm: null,
  birthdayDate: null,
  theme,
  // phoneNumber: {
  //   countryCode: COUNTRIES[0].code,
  //   number: null,
  //   fullPhoneNumber: null,
  // },
};

export const verifySignupInitialState: ISignupVerifyForm = {
  verifyCode: null,
};

export const forgotPasswordInitialState: IForgotPasswordForm = {
  identifierType: FORGOT_PASSWORD_IDENTIFIER_TYPE.EMAIL,
  identifier: null,
};

export const resetPasswordInitialState: IResetPasswordForm = {
  password: null,
  confirmPassword: null,
  resetToken: null,
};

export const googleSignupInitialState: IGoogleSignupForm = {
  idToken: null,
  username: null,
  // phoneNumber: {
  //   countryCode: COUNTRIES[0].code,
  //   number: null,
  //   fullPhoneNumber: null,
  // },
  gender: null,
  birthdayDate: null,
  theme,
};

```

## File: connectfy-client/src/modules/auth/ui/components/Footer/MainFooter/MainFooter.tsx
```typescript
import { Fragment, useCallback, useRef, useState } from "react";
import { useLocation } from "react-router-dom";
import { useTranslation } from "react-i18next";
import useBoolean from "@/hooks/useBoolean";
import { snack } from "@/common/utils/snackManager";
import { GoogleLogin } from "@react-oauth/google";
import SignupModal from "../../../Signup/components/SignupModal/SignupModal";
import { CHECK_UNIQUE_FIELD } from "@/common/enums/enums";
import { jwtDecode } from "jwt-decode";
import {
  useCheckUniqueMutation,
  useGoogleLoginMutation,
} from "@/modules/auth/api/api";
import { useErrors } from "@/hooks/useErrors";
import { useAuthStore } from "@/store/zustand/useAuthStore";
import { useTheme } from "@/context/ThemeContext";
import Button from "@/components/ui/CustomButton/Button/Button";
import GoogleIcon from "@/assets/icons/GoogleIcon";
import { ROUTER } from "@/common/constants/routet";
import { useAppNavigation } from "@/hooks/useAppNavigation";
import { getHomeRouteByStartup } from "@/common/utils/routes";

const MainFooter = () => {
  const { t } = useTranslation();
  const location = useLocation();
  const { toggleTheme } = useTheme();
  const { setToken } = useAuthStore();
  const { navigate } = useAppNavigation();

  const [checkUnique, { isLoading: LOADING_CHECK_UNIQUE }] =
    useCheckUniqueMutation();
  const [googleLogin] = useGoogleLoginMutation();

  const { showResponseErrors } = useErrors();

  const isModalOpen = useBoolean();

  const [idToken, setIdToken] = useState<string | null>(null);

  const googleButtonRef = useRef<HTMLDivElement>(null);
  const isLoginPage = location.pathname === ROUTER.AUTH.LOGIN;

  const handleNavigate = useCallback(() => {
    if (isLoginPage) {
      navigate(ROUTER.AUTH.SIGNUP);
    } else {
      navigate(`${ROUTER.AUTH.LOGIN}?method=username`);
    }
  }, [isLoginPage]);

  const handleGoogleSuccess = async (tokenResponse: any) => {
    try {
      const idToken = tokenResponse.credential;

      if (!idToken) {
        snack.error(t("error_messages.google_login_failed"));
        return;
      }

      if (isLoginPage) {
        try {
          const res = await googleLogin({ idToken }).unwrap();

          if (res.access_token) {
            setToken({
              type: "access_token",
              token: res.access_token,
            });
            snack.success(
              t("user_messages.login_successful", { lng: res.language }),
            );
            const redirectPage = getHomeRouteByStartup(res.startupPage);
            navigate(redirectPage);
            toggleTheme(res.theme);
          }
        } catch (error) {
          showResponseErrors(error);
        }
      } else {
        try {
          const decoded = jwtDecode<{ email?: string }>(idToken);
          const email = decoded.email;

          if (!email) {
            snack.error(t("error_messages.email_not_found"));
            return;
          }

          await checkUnique({
            field: CHECK_UNIQUE_FIELD.EMAIL,
            value: email,
          }).unwrap();
          setIdToken(idToken);
          isModalOpen.onOpen();
        } catch (error) {
          showResponseErrors(error);
        }
      }
    } catch (error: any) {
      snack.error(error);
    }
  };

  const handleCustomGoogleClick = () => {
    const googleButton = googleButtonRef.current?.querySelector(
      "div[role='button']",
    ) as HTMLElement;
    if (googleButton) {
      googleButton.click();
    }
  };

  return (
    <Fragment>
      {/* Social Login Separator */}
      <div className="relative py-2 my-5">
        <div className="absolute inset-0 flex items-center">
          <div className="w-full border-t border-(--input-border)"></div>
        </div>
        <div className="relative flex justify-center text-xs font-bold tracking-widest uppercase">
          <span className="bg-(--auth-main-bg) px-4 text-(--text-secondary)">
            {t("common.or_continue_with")}
          </span>
        </div>
      </div>

      {/* Social Buttons */}
      <div className="grid grid-cols-1 gap-4">
        {/* ✅ Custom Google Button */}
        <Button
          onClick={handleCustomGoogleClick}
          className="cursor-pointer flex items-center justify-center gap-3 py-3 rounded-xl border border-(--input-border) bg-(--input-bg) text-(--text-(--primary-color)) opacity-70 hover:opacity-100 duration-400 active:scale-[0.98] transition-all font-medium text-sm"
        >
          {/* Google SVG */}
          <GoogleIcon />
          Google
        </Button>

        {/* ✅ Real Google button (gizli) */}
        <div ref={googleButtonRef} className="hidden">
          <GoogleLogin
            onSuccess={handleGoogleSuccess}
            onError={() => snack.error(t("error_messages.google_login_failed"))}
            useOneTap={false}
          />
        </div>
      </div>

      <p className="text-center text-sm text-(--text-secondary) mb-5 mt-8">
        {isLoginPage
          ? t("common.do_not_have_account")
          : t("common.already_have_account")}{" "}
        <span
          className="font-bold text-(--primary-color) hover:underline cursor-pointer"
          onClick={handleNavigate}
        >
          {isLoginPage ? t("common.create_an_account") : t("common.login")}
        </span>
      </p>

      <SignupModal
        isOpen={isModalOpen.open}
        onClose={isModalOpen.onClose}
        idToken={idToken}
        isLoading={LOADING_CHECK_UNIQUE}
      />
    </Fragment>
  );
};

export default MainFooter;

```

## File: connectfy-client/src/modules/auth/ui/components/Footer/AuthFooter/AuthFooter.tsx
```typescript
import { LANGUAGE, THEME } from "@/common/enums/enums";
import SelectionModal from "@/components/Modal/SelectionModal/SelectionModal";
import Button from "@/components/ui/CustomButton/Button/Button";
import { useTheme } from "@/context/ThemeContext";
import useBoolean from "@/hooks/useBoolean";
import { Globe, Moon, Sun } from "lucide-react";
import { Fragment, useCallback } from "react";
import { useTranslation } from "react-i18next";

const AuthFooter = () => {
  const { i18n, t } = useTranslation();

  const screenWidth = window.innerWidth;

  const { theme, toggleTheme } = useTheme();
  const { open, onOpen, onClose } = useBoolean();

  const getButtonClass = (isActive: boolean) =>
    `cursor-pointer flex items-center justify-center rounded-xl duration-300 w-10 h-10 border transition-colors ${
      isActive
        ? "bg-[#34d399] border-[#34d399] text-white shadow-[0_4px_14px_0_rgba(52,211,153,0.39)]"
        : "bg-(--input-bg) border-(--input-border) text-(--text-secondary) hover:border-[#34d399]/30 hover:text-(--text-(--primary-color))"
    }`;

  const language = i18n.language;

  const languageList = [
    {
      name: "English",
      value: LANGUAGE.EN,
      icon: Globe,
      key: "EN",
      onClick: () => i18n.changeLanguage(LANGUAGE.EN),
    },
    {
      name: "Azərbaycan",
      value: LANGUAGE.AZ,
      icon: Globe,
      key: "AZ",
      onClick: () => i18n.changeLanguage(LANGUAGE.AZ),
    },
    {
      name: "Русский",
      value: LANGUAGE.RU,
      icon: Globe,
      key: "RU",
      onClick: () => i18n.changeLanguage(LANGUAGE.RU),
    },
    {
      name: "Türkçe",
      value: LANGUAGE.TR,
      icon: Globe,
      key: "TR",
      onClick: () => i18n.changeLanguage(LANGUAGE.TR),
    },
  ];

  const renderCurrentLanguage = useCallback(() => {
    const currentLanguage = languageList.find(
      (lang) => lang.value === language,
    );

    if (screenWidth < 1024) {
      return currentLanguage?.key;
    }

    return currentLanguage?.name;
  }, [language, screenWidth]);

  return (
    <Fragment>
      <div className="pt-8 pb-0 md:pb-8 flex items-center justify-between border-t border-(--input-border)">
        <div className="flex gap-4">
          <Button
            type="button"
            className="cursor-pointer flex items-center gap-1.5 text-xs font-medium text-(--text-secondary) hover:text-(--text-(--primary-color)) transition-colors"
            onClick={onOpen}
            tooltip={t("common.change_lang")}
            icon={<Globe size={15} />}
            title={renderCurrentLanguage()}
            hideTitleInMobile={false}
          />
        </div>
        <div className="flex gap-3 items-center">
          <Button
            type="button"
            className={getButtonClass(theme === THEME.LIGHT)}
            onClick={() => toggleTheme(THEME.LIGHT)}
            aria-pressed={theme === THEME.LIGHT}
            tooltip={t("common.light_theme")}
            icon={
              <Sun
                size={20}
                className={`${theme === THEME.LIGHT ? "font-bold" : ""}`}
              />
            }
          />
          <Button
            type="button"
            className={getButtonClass(theme === THEME.DARK)}
            onClick={() => toggleTheme(THEME.DARK)}
            aria-pressed={theme === THEME.DARK}
            tooltip={t("common.dark_theme")}
            icon={
              <Moon
                size={20}
                className={`${theme === THEME.DARK ? "font-bold" : ""}`}
              />
            }
          />
        </div>
      </div>

      <SelectionModal
        open={open}
        onClose={onClose}
        title={t("common.change_lang")}
        selections={languageList}
        activeKey={(language as string).toUpperCase()}
      />
    </Fragment>
  );
};

export default AuthFooter;

```

## File: connectfy-client/src/modules/auth/ui/Login/Login.tsx
```typescript
import { Fragment, useEffect } from "react";
import { IDENTIFIER_TYPE } from "@/common/enums/enums";
import { useFormik } from "formik";
import { loginInitialState } from "../../constants/intialState";
import { validateLogin } from "../../constants/validation";
import { useTranslation } from "react-i18next";
import { checkEmptyString } from "@/common/utils/checkValues";
import { ILoginForm, LoginModeType } from "../../types/types";
import { Link, useSearchParams } from "react-router-dom";
import { ROUTER } from "@/common/constants/routet";
import { snack } from "@/common/utils/snackManager";
import useFormDisabled from "@/hooks/useFormDisabled";
import PasswordInput from "@/components/ui/CustomInput/PasswordInput/PasswordInput";
import Input from "@/components/ui/CustomInput/Input/Input";
import PhoneNumberForm from "@/components/Form/PhoneNumberForm/PhoneNumberForm";
import Button from "@/components/ui/CustomButton/Button/Button";
import { useLoginMutation } from "@/modules/auth/api/api";
import { useErrors } from "@/hooks/useErrors";
import { useAuthStore } from "@/store/zustand/useAuthStore";
import { ShortcutTooltip } from "@/components/Tooltip/KeyboardShortcutTooltip";
import { useTheme } from "@/context/ThemeContext";
import MainFooter from "../components/Footer/MainFooter/MainFooter";
import { Mail, User } from "lucide-react";
import { useAppNavigation } from "@/hooks/useAppNavigation";
import { getHomeRouteByStartup } from "@/common/utils/routes";

const LOGIN_MODES: LoginModeType[] = ["username", "email", "phoneNumber"];

const Login = () => {
  const { t } = useTranslation();
  const { toggleTheme } = useTheme();
  const { setToken } = useAuthStore();
  const { navigate } = useAppNavigation();
  const { showResponseErrors } = useErrors();
  const [searchParams, setSearchParams] = useSearchParams();

  const rawLoginMode = searchParams.get("method");

  const method: LoginModeType = LOGIN_MODES.includes(
    rawLoginMode as LoginModeType,
  )
    ? (rawLoginMode as LoginModeType)
    : "username";

  const [login, { isLoading: LOADING_LOGIN }] = useLoginMutation();

  const formik = useFormik({
    initialValues: loginInitialState,
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    validate: (values) => validateLogin(values, t),
    onSubmit: async (values, { resetForm }) => {
      try {
        const res = await login(values).unwrap();

        if (res.isTwoFactorEnabled) {
          navigate(ROUTER.AUTH.VERIFY_LOGIN);
        } else if (res.access_token) {
          setToken({
            type: "access_token",
            token: res.access_token,
          });
          snack.success(
            t("user_messages.login_successful", { lng: res.language }),
          );
          const redirectPage = getHomeRouteByStartup(res.startupPage);
          navigate(redirectPage);
          toggleTheme(res.theme);
        }
        resetForm();
      } catch (err) {
        showResponseErrors(err);
      }
    },
  });

  const isDisabled = useFormDisabled<ILoginForm>({
    formik,
    loading: LOADING_LOGIN,
    validationRules: [
      (values) => !!values.identifier && checkEmptyString(values.identifier),
      (values) => {
        return checkEmptyString(values.password || "");
      },
    ],
  });

  const changeLoginMode = (mode: LoginModeType) => {
    formik.setFieldValue("identifier", "");
    setSearchParams({ method: mode }, { replace: true });
  };

  const renderMethodInput = () => {
    switch (method) {
      case "username":
        return (
          <Input
            className="w-full px-5 py-4 rounded-xl text-(--text-(--primary-color)) outline-none transition-all duration-200 placeholder:text-(--text-secondary)/50 focus:ring-2 focus:ring-[#34d399]/50"
            style={{
              backgroundColor: "var(--input-bg)",
              border: "1px solid var(--input-border)",
            }}
            title={t("common.username")}
            type="text"
            isFloating
            icon={<User size={20} />}
            name="identifier"
            onChange={formik.handleChange}
            onBlur={formik.handleBlur}
            value={formik.values.identifier || ""}
            isError={!!(formik.touched.identifier && formik.errors.identifier)}
            error={formik.errors.identifier}
            maxLength={30}
          />
        );
      case "email":
        return (
          <Input
            className="w-full px-5 py-4 rounded-xl text-(--text-(--primary-color)) outline-none transition-all duration-200 placeholder:text-(--text-secondary)/50 focus:ring-2 focus:ring-[#34d399]/50"
            style={{
              backgroundColor: "var(--input-bg)",
              border: "1px solid var(--input-border)",
            }}
            title={t("common.email")}
            type="text"
            isFloating
            icon={<Mail size={20} />}
            name="identifier"
            onChange={formik.handleChange}
            onBlur={formik.handleBlur}
            value={formik.values.identifier || ""}
            isError={!!(formik.touched.identifier && formik.errors.identifier)}
            error={formik.errors.identifier}
            maxLength={254}
          />
        );
      case "phoneNumber":
        return (
          <PhoneNumberForm
            name="phoneNumber"
            onChange={(value) =>
              formik.setFieldValue("identifier", value?.fullPhoneNumber)
            }
          />
        );
      default:
        return null;
    }
  };

  useEffect(() => {
    let identifierType: IDENTIFIER_TYPE;

    if (!rawLoginMode || !LOGIN_MODES.includes(rawLoginMode as LoginModeType)) {
      setSearchParams({ method: "username" }, { replace: true });
      identifierType = IDENTIFIER_TYPE.USERNAME;
    } else {
      switch (rawLoginMode) {
        case "email":
          identifierType = IDENTIFIER_TYPE.EMAIL;
          break;
        case "phoneNumber":
          identifierType = IDENTIFIER_TYPE.PHONE_NUMBER;
          break;
        default:
          identifierType = IDENTIFIER_TYPE.USERNAME;
          break;
      }
    }

    formik.setFieldValue("identifierType", identifierType);
  }, [rawLoginMode]);

  useEffect(() => {
    const handleSubmitEnter = (e: KeyboardEvent) => {
      if (e.key === "Enter" && !isDisabled) {
        e.preventDefault();
        formik.handleSubmit();
        return;
      }

      const modifierPressed = e.shiftKey && e.altKey;

      if (modifierPressed) {
        switch (e.code) {
          case "Digit1":
            e.preventDefault();
            changeLoginMode("username");
            break;
          case "Digit2":
            e.preventDefault();
            changeLoginMode("email");
            break;
          case "Digit3":
            e.preventDefault();
            changeLoginMode("phoneNumber");
            break;
        }
      }
    };

    document.addEventListener("keydown", handleSubmitEnter);
    return () => {
      document.removeEventListener("keydown", handleSubmitEnter);
    };
  }, [isDisabled, formik.handleSubmit]);

  return (
    <Fragment>
      {/* Form Header */}
      <div className="space-y-2 mb-5">
        <h2 className="text-3xl font-bold tracking-tight text-(--text-(--primary-color))">
          {t("common.welcome_back")}
        </h2>
        <p className="text-(--text-secondary) text-sm">
          {t("common.please_enter_your_details_to_sign_in_to_your_account")}
        </p>
      </div>

      {/* Login Mode Tabs */}
      <div className="flex border-b border-slate-100 my-4">
        <ShortcutTooltip keys={["Shift", "Alt", "1"]}>
          <Button
            type="button"
            className={`cursor-pointer px-4 py-2 w-full text-sm font-bold border-b-0 md:border-b-2 transition-colors ${
              method === "username"
                ? "text-(--primary-color) border-primary"
                : "text-slate-400 hover:text-slate-600 border-transparent"
            }`}
            onClick={() => changeLoginMode("username")}
          >
            {t("common.username")}
          </Button>
        </ShortcutTooltip>
        <ShortcutTooltip keys={["Shift", "Alt", "2"]}>
          <Button
            type="button"
            className={`cursor-pointer px-4 py-2 w-full text-sm font-medium border-b-0 md:border-b-2 transition-colors ${
              method === "email"
                ? "text-(--primary-color) border-primary"
                : "text-slate-400 hover:text-slate-600 border-transparent"
            }`}
            onClick={() => changeLoginMode("email")}
          >
            {t("common.email")}
          </Button>
        </ShortcutTooltip>
        <ShortcutTooltip keys={["Shift", "Alt", "3"]}>
          <Button
            type="button"
            className={`cursor-pointer px-4 py-2 w-full text-sm font-medium border-b-2 transition-colors ${
              method === "phoneNumber"
                ? "text-(--primary-color) border-primary"
                : "text-slate-400 hover:text-slate-600 border-transparent"
            }`}
            onClick={() => changeLoginMode("phoneNumber")}
          >
            {t("common.phoneNumber")}
          </Button>
        </ShortcutTooltip>
      </div>

      {/* Login Form */}
      <form className="space-y-6" onSubmit={formik.handleSubmit}>
        {/* Identifier Input */}
        <div className="space-y-2">{renderMethodInput()}</div>

        {/* Password Input */}
        <div className="space-y-2">
          <PasswordInput
            className="w-full px-5 py-4 rounded-xl text-(--text-(--primary-color)) outline-none transition-all duration-200 placeholder:text-(--text-secondary)/50 focus:ring-2 focus:ring-[#34d399]/50"
            style={{
              backgroundColor: "var(--input-bg)",
              border: "1px solid var(--input-border)",
            }}
            title={t("common.password")}
            isFloating
            name="password"
            onChange={formik.handleChange}
            onBlur={formik.handleBlur}
            value={formik.values.password || ""}
            isError={!!(formik.touched.password && formik.errors.password)}
            error={formik.errors.password}
            maxLength={30}
          />
          <div className="flex justify-end items-center ml-1">
            <Link
              to={ROUTER.AUTH.FORGOT_PASSWORD}
              className="text-xs font-bold text-[#34d399] hover:underline"
            >
              {t("common.forgot_password")}
            </Link>
          </div>
        </div>

        {/* Submit Button */}
        <Button
          className="duration-400 h-[60px] w-full py-4 font-bold text-lg rounded-xl transition-all transform active:scale-[0.98] hover:brightness-110 shadow-[0_4px_14px_0_rgba(52,211,153,0.39)] bg-(--primary-color) text-[#020a06]"
          type="submit"
          disabled={isDisabled}
          title={t("common.login")}
        />
      </form>

      <MainFooter />
    </Fragment>
  );
};

export default Login;

```

## File: connectfy-client/src/modules/auth/ui/ResesPassword/ResetPassword.tsx
```typescript
import { useTranslation } from "react-i18next";
import Button from "@/components/ui/CustomButton/Button/Button";
import PasswordInput from "@/components/ui/CustomInput/PasswordInput/PasswordInput.tsx";
import { useFormik } from "formik";
import { resetPasswordInitialState } from "../../constants/intialState";
import { validateResetPassword } from "../../constants/validation";
import { Fragment, useEffect } from "react";
import { TOKEN_TYPE } from "@/common/enums/enums";
import { useSearchParams } from "react-router-dom";
import { snack } from "@/common/utils/snackManager";
import useFormDisabled from "@/hooks/useFormDisabled";
import { IResetPasswordForm } from "../../types/types";
import { ROUTER } from "@/common/constants/routet";
import {
  useIsValidTokenMutation,
  useResetPasswordMutation,
} from "../../api/api";
import { useErrors } from "@/hooks/useErrors";
import Modal from "@/components/Modal";
import Spinner from "@/components/Spinner/Spinner";
import { useAppNavigation } from "@/hooks/useAppNavigation";

const ResetPassword = () => {
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();
  const { showFormikErrors } = useErrors();

  const [resetPassword, { isLoading: LOADING_RESET_PASSWORD }] =
    useResetPasswordMutation();
  const [isValidToken, { isLoading: LOADING_IS_VALID_TOKEN }] =
    useIsValidTokenMutation();

  const { showResponseErrors } = useErrors();

  const [searchParams] = useSearchParams();
  const token = searchParams.get("token");

  const formik = useFormik({
    initialValues: resetPasswordInitialState,
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    validate: (values) => validateResetPassword(values, t),
    onSubmit: async (values, { resetForm }) => {
      try {
        await resetPassword(values).unwrap();
        snack.success(t("user_messages.reset_password_successfull"));
        navigate("/auth");
        resetForm();
      } catch (error) {
        showResponseErrors(error);
      }
    },
  });

  const isDisabled = useFormDisabled<IResetPasswordForm>({
    formik,
    loading: LOADING_RESET_PASSWORD,
    validationRules: [
      (values) => !!values.password,
      (values) => !!values.confirmPassword,
      (values) => !!values.resetToken,
      !!token,
    ],
  });

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    const errors = await formik.validateForm();
    if (Object.keys(errors).length > 0) {
      showFormikErrors(errors);
      return;
    }
    formik.handleSubmit(e as any);
  };

  useEffect(() => {
    if (!token) {
      navigate(ROUTER.AUTH.LOGIN);
      return;
    }

    (async () => {
      try {
        const res = await isValidToken({
          token,
          type: TOKEN_TYPE.PASSWORD_RESET,
        }).unwrap();

        if (!res) {
          navigate(ROUTER.AUTH.LOGIN);
          return;
        }
      } catch {
        navigate(ROUTER.AUTH.LOGIN);
      }
    })();

    formik.setFieldValue("resetToken", token);
  }, [token]);

  useEffect(() => {
    if (isDisabled) return;

    const handleSubmitEnter = (e: KeyboardEvent) => {
      if (e.key === "Enter") {
        e.preventDefault();
        handleSubmit(e as any);
      }
    };

    document.addEventListener("keydown", handleSubmitEnter);
    return () => {
      document.removeEventListener("keydown", handleSubmitEnter);
    };
  }, [isDisabled, formik.handleSubmit]);

  return (
    <Fragment>
      {LOADING_IS_VALID_TOKEN ? (
        <Modal open={LOADING_IS_VALID_TOKEN} onClose={() => {}}>
          <div className="flex flex-col items-center justify-center gap-8">
            <Spinner size={30} style={{ color: "var(--primary-color)" }} />

            <p className="font-semibold lg:text-xl text-lg text-(--text-primary)">
              {t("common.checking")}
            </p>
          </div>
        </Modal>
      ) : (
        <Fragment>
          <div className="w-full">
            <div className="mb-10 text-center md:text-left">
              <h2 className="text-3xl font-bold mb-3 text-(--text-primary)">
                {t("common.reset_password_title")}
              </h2>
            </div>
          </div>

          <form className="space-y-8" onSubmit={handleSubmit}>
            <div className="space-y-4">
              <PasswordInput
                title={t("common.password")}
                name="password"
                className="w-full px-4 py-3 rounded-xl border border-slate-200 dark:border-input-border bg-white dark:bg-form-bg text-slate-900 dark:text-white focus:ring-2 focus:ring-primary outline-none"
                showGenerateButton
                isFloating
                value={formik.values.password || ""}
                onChange={(e) => {
                  const value = e.target.value || null;

                  if (value && value.length > 30) return;

                  formik.setFieldValue("password", value || null);
                }}
                onBlur={() => formik.setFieldTouched("password", true, false)}
                onGenerate={(value) => {
                  formik.setFieldValue("password", value);
                  formik.setFieldValue("confirmPassword", value);
                }}
                isError={!!formik.errors.password}
                maxLength={30}
              />
              <PasswordInput
                title={t("common.confirm_password")}
                name="confirmPassword"
                className="w-full px-4 py-3 rounded-xl border border-slate-200 dark:border-input-border bg-white dark:bg-form-bg text-slate-900 dark:text-white focus:ring-2 focus:ring-primary outline-none"
                value={formik.values.confirmPassword || ""}
                isFloating
                onChange={(e) => {
                  const value = e.target.value || null;

                  if (value && value.length > 30) return;

                  formik.setFieldValue("confirmPassword", value || null);
                }}
                onBlur={() =>
                  formik.setFieldTouched("confirmPassword", true, false)
                }
                isError={!!formik.errors.confirmPassword}
                maxLength={30}
              />
            </div>
            <div className="flex flex-col items-center gap-4">
              <Button
                className="duration-400 h-[60px] w-full py-4 font-bold text-lg rounded-xl transition-all transform active:scale-[0.98] hover:brightness-110 shadow-[0_4px_14px_0_rgba(52,211,153,0.39)] bg-(--primary-color) text-[#020a06]"
                type="submit"
                disabled={isDisabled}
                isLoading={LOADING_RESET_PASSWORD}
                title={t("common.submit")}
              />
            </div>
          </form>
        </Fragment>
      )}
    </Fragment>
  );
};

export default ResetPassword;

```

## File: connectfy-client/src/modules/auth/ui/VerifySignup/VerifySignup.tsx
```typescript
import { useTranslation } from "react-i18next";
import { LOCAL_STORAGE_KEYS } from "@/common/enums/enums";
import { useFormik } from "formik";
import { verifySignupInitialState } from "../../constants/intialState";
import { validateVerifySignup } from "../../constants/validation";
import { useCallback, useEffect, useState } from "react";
import { onPressEnter, onPressEsc } from "@/common/utils/keyPressDown";
import { ROUTER } from "@/common/constants/routet";
import { snack } from "@/common/utils/snackManager";
import OTPForm from "@/components/Form/OTPForm/OTPForm";
import Button from "@/components/ui/CustomButton/Button/Button";
import { formatTimeToSeconds } from "@/common/utils/formatValues";
import {
  useResendSignupVerifyMutation,
  useSignupVerifyMutation,
} from "../../api/api";
import { ISignupForm, ISignupVerifyForm } from "../../types/types";
import useFormDisabled from "@/hooks/useFormDisabled";
import { useErrors } from "@/hooks/useErrors";
import { useAuthStore } from "@/store/zustand/useAuthStore";
import { ArrowLeft } from "lucide-react";
import { useAppNavigation } from "@/hooks/useAppNavigation";

const TIMER_DURATION = 60;

const VerifySignup = () => {
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();

  const signupForm = JSON.parse(
    localStorage.getItem(LOCAL_STORAGE_KEYS.SIGNUP_FORM) || "{}",
  ) as ISignupForm | null;

  const [signupVerify, { isLoading: LOADING_SIGNUP_VERIFY }] =
    useSignupVerifyMutation();
  const [resendSignupVerify, { isLoading: LOADING_RESEND_SIGNUP_VERIFY }] =
    useResendSignupVerifyMutation();

  const { setToken } = useAuthStore();
  const { showResponseErrors } = useErrors();

  const [timeLeft, setTimeLeft] = useState<number>(TIMER_DURATION);
  const [isResendDisabled, setIsResendDisabled] = useState<boolean>(true);

  const formik = useFormik({
    initialValues: verifySignupInitialState,
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    validate: (values) => validateVerifySignup(values, t),
    onSubmit: async (values, { resetForm }) => {
      try {
        const res = await signupVerify(values).unwrap();
        if (res.access_token) {
          setToken({
            type: "access_token",
            token: res.access_token,
          });
          snack.success(t("user_messages.verify_successful"));
          navigate(ROUTER.MESSENGER.MAIN);
        }
        localStorage.removeItem(LOCAL_STORAGE_KEYS.OTP_EXPIRES_AT);
        localStorage.removeItem(LOCAL_STORAGE_KEYS.SIGNUP_FORM);
        resetForm();
      } catch (error) {
        if ((error as any)?.additional?.navigate) {
          navigate(`${ROUTER.AUTH.LOGIN}?method=username`);
        } else {
          showResponseErrors(error);
        }
      }
    },
  });

  const onKeyDown = (e: React.KeyboardEvent<HTMLElement>) => {
    switch (e.key) {
      case "Enter":
        onPressEnter(e, () => {
          if (isDisabled) return;
          formik.handleSubmit();
        });
        break;

      case "Escape":
        onPressEsc(e, () => navigate(-1));
    }
  };

  const handleOtpChange = useCallback(
    (value: string | null) => {
      const current = formik.values.verifyCode ?? "";
      const next = value ?? "";
      if (current !== next) {
        formik.setFieldValue("verifyCode", next);
      }
    },
    [formik],
  );

  const isDisabled = useFormDisabled<ISignupVerifyForm>({
    formik,
    loading: LOADING_SIGNUP_VERIFY,
    validationRules: [
      (values) =>
        !!(
          values.verifyCode &&
          values.verifyCode.length === 6 &&
          values.verifyCode.trim() !== ""
        ),
    ],
  });

  const startTimer = () => {
    const expiresAt = Date.now() + TIMER_DURATION * 1000;
    localStorage.setItem(
      LOCAL_STORAGE_KEYS.OTP_EXPIRES_AT,
      expiresAt.toString(),
    );
    setTimeLeft(TIMER_DURATION);
    setIsResendDisabled(true);
  };

  const handleResendCode = async () => {
    if (isResendDisabled) return;

    try {
      await resendSignupVerify().unwrap();
      snack.success(t("user_messages.otp_resent"));
      startTimer();
    } catch (error) {
      if ((error as any)?.additional?.navigate) {
        navigate(`${ROUTER.AUTH.LOGIN}?method=username`);
      } else {
        showResponseErrors(error);
      }
    }
  };

  useEffect(() => {
    const storedExpiresAt = localStorage.getItem(
      LOCAL_STORAGE_KEYS.OTP_EXPIRES_AT,
    );

    if (!storedExpiresAt) {
      startTimer();
    } else {
      const difference = Math.floor(
        (parseInt(storedExpiresAt) - Date.now()) / 1000,
      );

      if (difference > 0) {
        setTimeLeft(difference);
        setIsResendDisabled(true);
      } else {
        setTimeLeft(0);
        setIsResendDisabled(false);
        localStorage.removeItem(LOCAL_STORAGE_KEYS.OTP_EXPIRES_AT);
      }
    }
  }, []);

  useEffect(() => {
    if (timeLeft <= 0) {
      setIsResendDisabled(false);
      localStorage.removeItem(LOCAL_STORAGE_KEYS.OTP_EXPIRES_AT);
      return;
    }

    const interval = setInterval(() => {
      const storedExpiresAt = localStorage.getItem(
        LOCAL_STORAGE_KEYS.OTP_EXPIRES_AT,
      );
      if (storedExpiresAt) {
        const difference = Math.floor(
          (parseInt(storedExpiresAt) - Date.now()) / 1000,
        );

        if (difference > 0) {
          setTimeLeft(difference);
        } else {
          setTimeLeft(0);
          setIsResendDisabled(false);
          localStorage.removeItem(LOCAL_STORAGE_KEYS.OTP_EXPIRES_AT);
        }
      }
    }, 1000);

    return () => clearInterval(interval);
  }, [timeLeft]);

  return (
    <div className="w-full">
      <div className="mb-10 text-center md:text-left">
        <h2 className="text-3xl font-bold mb-3 text-(--text-primary)">
          {t("common.verify_account_heading")}
        </h2>
        <p className="text-(--text-primary)">
          {t("common.verify_account_message_part_1")}:{" "}
          <span className="text-(--primary-color) font-bold">
            {signupForm?.email ?? "example@gmail.com"}
          </span>
          . {t("common.verify_account_message_part_2")}
        </p>
      </div>
      <form className="space-y-8" onSubmit={formik.handleSubmit}>
        <OTPForm
          name="verifyCode"
          length={6}
          onChange={handleOtpChange}
          onKeyDown={onKeyDown}
          onComplete={(code) => formik.setFieldValue("verifyCode", code)}
        />
        <div className="flex flex-col items-center gap-4">
          <p className="text-sm text-slate-500 dark:text-slate-400">
            {t("common.did_not_receive_code")}{" "}
            <Button
              type="button"
              onClick={handleResendCode}
              disabled={isResendDisabled}
              isLoading={LOADING_RESEND_SIGNUP_VERIFY}
              className={`font-medium transition-all bg-transparent border-none ${
                isResendDisabled
                  ? "text-slate-400"
                  : "text-(--primary-color) hover:underline"
              }`}
              title={t("common.resend_code")}
            />
            {isResendDisabled && (
              <span className="text-slate-400 ml-2 font-mono">
                ({formatTimeToSeconds(timeLeft)})
              </span>
            )}
          </p>
          <Button
            className="duration-400 h-[60px] w-full py-4 font-bold text-lg rounded-xl transition-all transform active:scale-[0.98] hover:brightness-110 shadow-[0_4px_14px_0_rgba(52,211,153,0.39)] bg-(--primary-color) text-[#020a06]"
            type="submit"
            disabled={isDisabled}
            isLoading={LOADING_SIGNUP_VERIFY}
            title={t("common.verify")}
          />
          <Button
            className="flex items-center gap-1 text-sm text-slate-500 dark:text-slate-400 hover:text-primary transition-colors mt-2"
            onClick={() => navigate(ROUTER.AUTH.SIGNUP)}
          >
            <ArrowLeft size={20} />
            {t("common.back")}
          </Button>
        </div>
      </form>
    </div>
  );
};

export default VerifySignup;

```

## File: connectfy-client/src/modules/auth/ui/VerifyLogin/VerifyLogin.tsx
```typescript
import { useTranslation } from "react-i18next";
import { useFormik } from "formik";
import { loginVerifyInitialState } from "../../constants/intialState";
import { validateVerifyLogin } from "../../constants/validation";
import { useCallback } from "react";
import { onPressEnter, onPressEsc } from "@/common/utils/keyPressDown";
import { ROUTER } from "@/common/constants/routet";
import { snack } from "@/common/utils/snackManager";
import OTPForm from "@/components/Form/OTPForm/OTPForm";
import Button from "@/components/ui/CustomButton/Button/Button";
import { useLoginVerifyMutation } from "../../api/api";
import { ILoginVerifyForm } from "../../types/types";
import useFormDisabled from "@/hooks/useFormDisabled";
import { useErrors } from "@/hooks/useErrors";
import { useTheme } from "@/context/ThemeContext";
import { useAuthStore } from "@/store/zustand/useAuthStore";
import { ArrowLeft } from "lucide-react";
import { useAppNavigation } from "@/hooks/useAppNavigation";
import { getHomeRouteByStartup } from "@/common/utils/routes";

const VerifyLogin = () => {
  const { t } = useTranslation();
  const { toggleTheme } = useTheme();
  const { setToken } = useAuthStore();
  const { navigate } = useAppNavigation();

  const [loginVerify, { isLoading }] = useLoginVerifyMutation();

  const { showResponseErrors } = useErrors();

  const formik = useFormik({
    initialValues: loginVerifyInitialState,
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    validate: (values) => validateVerifyLogin(values, t),
    onSubmit: async (values, { resetForm }) => {
      try {
        const res = await loginVerify(values).unwrap();
        if (res.access_token) {
          setToken({
            type: "access_token",
            token: res.access_token,
          });
          snack.success(
            t("user_messages.login_successful", { lng: res.language }),
          );
          const redirectPage = getHomeRouteByStartup(res.startupPage);
          navigate(redirectPage);
          toggleTheme(res.theme);
        }
        resetForm();
      } catch (error) {
        if ((error as any)?.additional?.navigate) {
          navigate(`${ROUTER.AUTH.LOGIN}?method=username`);
        } else {
          showResponseErrors(error);
        }
      }
    },
  });

  const onKeyDown = (e: React.KeyboardEvent<HTMLElement>) => {
    switch (e.key) {
      case "Enter":
        onPressEnter(e, () => {
          if (isDisabled) return;
          formik.handleSubmit();
        });
        break;

      case "Escape":
        onPressEsc(e, () => navigate(-1));
    }
  };

  const handleOtpChange = useCallback(
    (value: string | null) => {
      const current = formik.values.code ?? "";
      const next = value ?? "";
      if (current !== next) {
        formik.setFieldValue("code", next);
      }
    },
    [formik],
  );

  const isDisabled = useFormDisabled<ILoginVerifyForm>({
    formik,
    loading: isLoading,
    validationRules: [
      (values) =>
        !!(
          values.code &&
          values.code.length === 6 &&
          values.code.trim() !== ""
        ),
    ],
  });

  return (
    <div className="w-full">
      <div className="mb-10 text-center md:text-left">
        <h2 className="text-3xl font-bold mb-3 text-(--text-primary)">
          {t("common.verify_account_heading")}
        </h2>
        <p className="text-(--text-primary)">
          {t("common.verify_account_message_part_1")}.{" "}
          {t("common.verify_account_message_part_2")}
        </p>
      </div>
      <form className="space-y-8" onSubmit={formik.handleSubmit}>
        <OTPForm
          name="code"
          length={6}
          onChange={handleOtpChange}
          onKeyDown={onKeyDown}
          onComplete={(code) => formik.setFieldValue("code", code)}
        />
        <div className="flex flex-col items-center gap-4">
          <Button
            className="duration-400 h-[60px] w-full py-4 font-bold text-lg rounded-xl transition-all transform active:scale-[0.98] hover:brightness-110 shadow-[0_4px_14px_0_rgba(52,211,153,0.39)] bg-(--primary-color) text-[#020a06]"
            type="submit"
            disabled={isDisabled}
            isLoading={isLoading}
            title={t("common.verify")}
          />
          <Button
            className="flex items-center gap-1 text-sm text-slate-500 dark:text-slate-400 hover:text-primary transition-colors mt-2"
            onClick={() => navigate(-1)}
          >
            <ArrowLeft size={20} />
            {t("common.back")}
          </Button>
        </div>
      </form>
    </div>
  );
};

export default VerifyLogin;

```

## File: connectfy-client/src/modules/auth/ui/Main/Main.tsx
```typescript
import { useEffect, FC } from "react";
import { useSearchParams } from "react-router-dom";
import { ROUTER } from "@/common/constants/routet";
import { snack } from "@/common/utils/snackManager";
import { useTranslation } from "react-i18next";
import MainSpinner from "@/components/Loading/Loading";
import { useRestoreAccountMutation } from "../../api/api";
import { useErrors } from "@/hooks/useErrors";
import { useAuthStore } from "@/store/zustand/useAuthStore";
import { useTheme } from "@/context/ThemeContext";
import { useAppNavigation } from "@/hooks/useAppNavigation";

const MainPage: FC = () => {
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();

  const [restoreAccount, { isLoading: LOADING_RESTORE_ACCOUNT }] =
    useRestoreAccountMutation();

  const { toggleTheme } = useTheme();
  const { setToken } = useAuthStore();
  const { showResponseErrors } = useErrors();

  const [searchParams] = useSearchParams();
  const token = searchParams.get("token");
  const type = searchParams.get("type");

  useEffect(() => {
    if (!token || !type || type !== "restore") {
      navigate(ROUTER.AUTH.LOGIN);
      return;
    }

    (async () => {
      try {
        const res = await restoreAccount({ token }).unwrap();

        setToken({
          type: "access_token",
          token: res.access_token,
        });
        navigate(ROUTER.MAIN);
        snack.success(t("user_messages.restore_account_successful"));
        navigate(res.startupPage);
        toggleTheme(res.theme);
      } catch (error) {
        showResponseErrors(error);
        navigate(ROUTER.AUTH.LOGIN);
      }
    })();
  }, [navigate, t, token, type]);

  return (
    <section>
      {LOADING_RESTORE_ACCOUNT ? (
        <MainSpinner description={{ title: t("common.restoring_account") }} />
      ) : null}
    </section>
  );
};

export default MainPage;

```

## File: connectfy-client/src/modules/auth/ui/Signup/Signup.tsx
```typescript
import Input from "@/components/ui/CustomInput/Input/Input.tsx";
import PasswordInput from "@/components/ui/CustomInput/PasswordInput/PasswordInput.tsx";
import { useTranslation } from "react-i18next";
import { Link } from "react-router-dom";
import { useFormik } from "formik";
import { signupInitialState } from "../../constants/intialState";
import { validateSignup } from "../../constants/validation";
import { GENDER, LOCAL_STORAGE_KEYS, THEME } from "@/common/enums/enums";
import { Fragment, useEffect, useState } from "react";
import { checkEmptyString } from "@/common/utils/checkValues";
import { ROUTER } from "@/common/constants/routet";
import useFormDisabled from "@/hooks/useFormDisabled";
import { ISignupForm } from "@/modules/auth/types/types";
import Checkbox from "@/components/ui/CustomCheckbox/Checkbox/Checkbox";
import DatePicker from "@/components/ui/DatePicker/DatePicker";
import Button from "@/components/ui/CustomButton/Button/Button";
import { useSignupMutation } from "@/modules/auth/api/api";
import { useErrors } from "@/hooks/useErrors";
import MainFooter from "../components/Footer/MainFooter/MainFooter";
import { Mail, User } from "lucide-react";
import { useAppNavigation } from "@/hooks/useAppNavigation";

const Signup = () => {
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();
  const { showFormikErrors } = useErrors();

  const signupForm = JSON.parse(
    localStorage.getItem(LOCAL_STORAGE_KEYS.SIGNUP_FORM) || "{}",
  ) as ISignupForm | null;
  const [signup, { isLoading: LOADING_SIGNUP }] = useSignupMutation();

  const { showResponseErrors } = useErrors();

  const [checked, setChecked] = useState<boolean>(false);

  const formik = useFormik({
    initialValues: signupForm || signupInitialState,
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    validate: (values) => validateSignup(values, t),
    onSubmit: async (values, { resetForm }) => {
      try {
        const { confirm, ...rest } = values;
        void confirm;
        rest.birthdayDate = rest.birthdayDate
          ? new Date(rest.birthdayDate)
          : null;

        rest.theme =
          (localStorage.getItem(LOCAL_STORAGE_KEYS.APP_THEME) as THEME) ||
          THEME.LIGHT;

        await signup(rest).unwrap();
        resetForm();
        localStorage.setItem(
          LOCAL_STORAGE_KEYS.SIGNUP_FORM,
          JSON.stringify(values),
        );
        navigate(ROUTER.AUTH.VERIFY_SIGNUP, { replace: true });
      } catch (error) {
        showResponseErrors(error);
      }
    },
  });

  const isDisabled = useFormDisabled<ISignupForm>({
    formik,
    loading: LOADING_SIGNUP,
    validationRules: [
      (values) => {
        const stringFields = [
          values.firstName,
          values.lastName,
          values.username,
          values.email,
          values.password,
          values.confirm,
        ];
        return !stringFields.some((v) => !v || !checkEmptyString(v));
      },
      (values) => !!values.gender,
      (values) => !!values.birthdayDate,
      checked,
    ],
  });

  const selectGender = (gender: GENDER) => {
    if (formik.values.gender === gender) {
      formik.setFieldValue("gender", null);
    } else {
      formik.setFieldValue("gender", gender);
    }
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    const errors = await formik.validateForm();
    if (Object.keys(errors).length > 0) {
      showFormikErrors(errors);
      return;
    }
    formik.handleSubmit(e as any);
  };

  useEffect(() => {
    if (isDisabled) return;

    const handleSubmitEnter = (e: KeyboardEvent) => {
      if (e.key === "Enter") {
        e.preventDefault();
        handleSubmit(e as any);
      }
    };

    document.addEventListener("keydown", handleSubmitEnter);
    return () => {
      document.removeEventListener("keydown", handleSubmitEnter);
    };
  }, [isDisabled, formik]);

  return (
    <Fragment>
      <div className="space-y-2 mb-5">
        <h2 className="text-3xl font-bold tracking-tight text-(--text-(--primary-color))">
          {t("common.join_connectfy")}
        </h2>
        <p className="text-(--text-secondary) text-sm">
          {t("common.join_connectfy_description")}
        </p>
      </div>
      <form className="space-y-4" onSubmit={handleSubmit}>
        <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
          <div className="space-y-1.5">
            <Input
              className="w-full px-4 py-3 rounded-xl border border-slate-200 dark:border-input-border bg-white dark:bg-form-bg text-slate-900 dark:text-white placeholder:text-slate-400 dark:placeholder:text-[#99c2aa]/30 focus:ring-2 focus:ring-primary outline-none"
              type="text"
              name="firstName"
              title={t("common.first_name")}
              isFloating
              icon={<User size={20} />}
              value={formik.values.firstName || ""}
              onChange={formik.handleChange}
              isError={!!formik.errors.firstName}
              maxLength={50}
            />
          </div>
          <div className="space-y-1.5">
            <Input
              className="w-full px-4 py-3 rounded-xl border border-slate-200 dark:border-input-border bg-white dark:bg-form-bg text-slate-900 dark:text-white placeholder:text-slate-400 dark:placeholder:text-[#99c2aa]/30 focus:ring-2 focus:ring-primary outline-none"
              type="text"
              name="lastName"
              title={t("common.last_name")}
              isFloating
              icon={<User size={20} />}
              value={formik.values.lastName || ""}
              onChange={formik.handleChange}
              isError={!!formik.errors.lastName}
              maxLength={50}
            />
          </div>
        </div>
        <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
          <div className="space-y-1.5">
            <Input
              className="w-full px-4 py-3 rounded-xl border border-slate-200 dark:border-input-border bg-white dark:bg-form-bg text-slate-900 dark:text-white placeholder:text-slate-400 dark:placeholder:text-[#99c2aa]/30 focus:ring-2 focus:ring-primary outline-none"
              type="text"
              name="username"
              title={t("common.username")}
              isFloating
              icon={<User size={20} />}
              value={formik.values.username || ""}
              onChange={formik.handleChange}
              isError={!!formik.errors.username}
              maxLength={30}
            />
          </div>
          <div className="space-y-1.5">
            <Input
              className="w-full px-4 py-3 rounded-xl border border-slate-200 dark:border-input-border bg-white dark:bg-form-bg text-slate-900 dark:text-white placeholder:text-slate-400 dark:placeholder:text-[#99c2aa]/30 focus:ring-2 focus:ring-primary outline-none"
              type="email"
              name="email"
              title={t("common.email")}
              isFloating
              icon={<Mail size={20} />}
              value={formik.values.email || ""}
              onChange={formik.handleChange}
              isError={!!formik.errors.email}
              maxLength={254}
            />
          </div>
        </div>
        <div className="space-y-1.5">
          <DatePicker
            value={formik.values.birthdayDate?.toString() || ""}
            onChange={(date) => formik.setFieldValue("birthdayDate", date)}
            inputSize="large"
            title={t("common.birthday")}
            hasError={
              !!(formik.errors.birthdayDate && formik.touched.birthdayDate)
            }
          />
        </div>
        <div className="grid grid-cols-3 gap-3">
          {Object.keys(GENDER).map((gender) => (
            <Button
              key={gender}
              className={`cursor-pointer py-4 px-2 border border-(--input-border) rounded-lg text-sm font-medium hover:border-primary transition-colors text-(--text-secondary) duration-900 ${formik.values.gender === gender ? "bg-(--primary-color) text-white" : ""}`}
              type="button"
              onClick={() => selectGender(gender as GENDER)}
              title={t(`enum.${gender.toLowerCase()}`)}
            />
          ))}
        </div>
        <div className="space-y-4">
          <div className="space-y-1.5">
            <div className="relative">
              <PasswordInput
                className="w-full px-4 py-3 rounded-xl border border-slate-200 dark:border-input-border bg-white dark:bg-form-bg text-slate-900 dark:text-white focus:ring-2 focus:ring-primary outline-none"
                type="password"
                title={t("common.password")}
                isFloating
                showGenerateButton
                onGenerate={(value) => {
                  formik.setFieldValue("password", value);
                  formik.setFieldValue("confirm", value);
                }}
                name="password"
                value={formik.values.password || ""}
                onChange={formik.handleChange}
                isError={!!formik.errors.password}
                maxLength={30}
              />
            </div>
          </div>
          <div className="space-y-1.5">
            <div className="relative">
              <PasswordInput
                className="w-full px-4 py-3 rounded-xl border border-slate-200 dark:border-input-border bg-white dark:bg-form-bg text-slate-900 dark:text-white focus:ring-2 focus:ring-primary outline-none"
                type="password"
                title={t("common.confirm_password")}
                isFloating
                name="confirm"
                value={formik.values.confirm || ""}
                onChange={formik.handleChange}
                isError={!!formik.errors.confirm}
                maxLength={30}
              />
            </div>
          </div>
        </div>
        <div className="py-1">
          <Checkbox
            id="terms"
            checked={checked}
            onChange={() => setChecked(!checked)}
          >
            {t("common.terms_prefix")}{" "}
            <Link
              to={ROUTER.TERMS_AND_CONDITIONS}
              className="text-(--primary-color) underline cursor-pointer"
              target="_blank"
            >
              {t("common.terms_link")}
            </Link>{" "}
            {t("common.terms_suffix")}
          </Checkbox>
        </div>
        <Button
          className="duration-400 h-[60px] w-full py-4 font-bold text-lg rounded-xl transition-all transform active:scale-[0.98] hover:brightness-110 shadow-[0_4px_14px_0_rgba(52,211,153,0.39)] bg-(--primary-color) text-[#020a06]"
          type="submit"
          disabled={isDisabled}
          isLoading={LOADING_SIGNUP}
          title={t("common.signup")}
        />
      </form>

      <MainFooter />
    </Fragment>
  );
};

export default Signup;

```

## File: connectfy-client/src/modules/auth/ui/Signup/components/SignupModal/SignupModal.tsx
```typescript
import { useState, useEffect } from "react";
import { useTranslation } from "react-i18next";
import { useFormik } from "formik";
import { googleSignupInitialState } from "../../../../constants/intialState";
import { valdiateGoogleSignup } from "../../../../constants/validation";
import { GENDER, LOCAL_STORAGE_KEYS, THEME } from "@/common/enums/enums";
import { checkEmptyString } from "@/common/utils/checkValues";
import { ROUTER } from "@/common/constants/routet";
import { snack } from "@/common/utils/snackManager";
import { onPressEnter, onPressEsc } from "@/common/utils/keyPressDown";
import { useGoogleSignupMutation } from "@/modules/auth/api/api";

// MUI Components (Sadece loading üçün)
import Modal from "@/components/Modal";
import Input from "@/components/ui/CustomInput/Input/Input";
import CustomDatePicker from "@/components/ui/DatePicker/DatePicker";
import Button from "@/components/ui/CustomButton/Button/Button";
import Spinner from "@/components/Spinner/Spinner";

// Hooks
import { useErrors } from "@/hooks/useErrors";
import { useAuthStore } from "@/store/zustand/useAuthStore";
import { User, UserRoundCheck } from "lucide-react";
import { useAppNavigation } from "@/hooks/useAppNavigation";

interface SignupModalProps {
  idToken: string | null;
  isOpen: boolean;
  onClose: () => void;
  isLoading: boolean;
}

const SignupModal = ({
  idToken,
  isOpen,
  onClose,
  isLoading,
}: SignupModalProps) => {
  const { t } = useTranslation();
  const { setToken } = useAuthStore();
  const { navigate } = useAppNavigation();

  const [isDisabled, setIsDisabled] = useState<boolean>(true);
  const [googleSignup, { isLoading: LOADING_GOOGLE_SIGNUP }] =
    useGoogleSignupMutation();

  const { showResponseErrors } = useErrors();

  const formik = useFormik({
    initialValues: googleSignupInitialState,
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    validate: (values) => valdiateGoogleSignup(values, t),
    onSubmit: async (values, { resetForm }) => {
      try {
        values.birthdayDate = values.birthdayDate
          ? new Date(values.birthdayDate)
          : null;

        values.theme =
          (localStorage.getItem(LOCAL_STORAGE_KEYS.APP_THEME) as THEME) ||
          THEME.LIGHT;

        const res = await googleSignup(values).unwrap();
        if (res.access_token) {
          snack.success(t("user_messages.signup_successful"));
          setToken({
            type: "access_token",
            token: res.access_token,
          });
          navigate(ROUTER.MESSENGER.MAIN);
        }
        resetForm();
        onClose();
      } catch (error) {
        showResponseErrors(error);
      }
    },
  });

  const onKeyDown = (e: React.KeyboardEvent<HTMLElement>) => {
    if (isLoading) return;

    switch (e.key) {
      case "Enter":
        onPressEnter(e, () => {
          if (isDisabled) return;
          formik.handleSubmit();
        });
        break;
      case "Escape":
        onPressEsc(e, onClose);
    }
  };

  const selectGender = (gender: GENDER) => {
    if (formik.values.gender === gender) {
      formik.setFieldValue("gender", null);
    } else {
      formik.setFieldValue("gender", gender);
    }
  };

  useEffect(() => {
    formik.setFieldValue("idToken", idToken);
  }, [idToken]);

  useEffect(() => {
    const { username, gender, birthdayDate } = formik.values;

    const hasEmptyUsername = !username || !checkEmptyString(username);
    const hasEmptyGender = !gender;
    const hasEmptyDate = !birthdayDate;

    const shouldDisable =
      hasEmptyUsername ||
      hasEmptyGender ||
      hasEmptyDate ||
      LOADING_GOOGLE_SIGNUP;

    setIsDisabled(shouldDisable);
  }, [formik.values, LOADING_GOOGLE_SIGNUP]);

  if (!isOpen) return null;

  return (
    <Modal open={isOpen} onClose={onClose}>
      {isLoading ? (
        <div className="flex flex-col items-center justify-center gap-8">
          <Spinner size={30} style={{ color: "var(--primary-color)" }} />

          <p className="font-semibold lg:text-xl text-lg text-(--text-primary)">
            {t("common.checking_email")}
          </p>
        </div>
      ) : (
        <div
          className="relative w-full max-w-[480px] rounded-[32px] border border-(--input-border) bg-(--auth-main-bg) p-8 shadow-2xl overflow-hidden"
          onClick={(e) => e.stopPropagation()}
          onKeyDown={onKeyDown}
        >
          {/* Glow Effects (Background Decorations) */}
          <div className="absolute top-0 left-1/2 -translate-x-1/2 w-64 h-32 bg-(--primary-color)/10 blur-[60px] rounded-full pointer-events-none" />

          {/* Header Section */}
          <div className="relative z-10 flex flex-col items-center text-center mb-8">
            <div className="mb-6 flex h-20 w-20 items-center justify-center rounded-2xl border border-(--input-border) bg-(--auth-main-bg) text-(--primary-color) shadow-[0_0_30px_-5px_rgba(52,211,153,0.3)]">
              <UserRoundCheck />
            </div>

            <h2 className="text-2xl font-bold text-(--text-primary) mb-2">
              {t("common.complete_signup")}
            </h2>
            <p className="text-(--muted-color) text-sm max-w-[80%] leading-relaxed">
              {t("common.google_signup_description")}
            </p>
          </div>

          {/* Form Content */}
          <form
            className="relative z-10 flex flex-col gap-5"
            onSubmit={formik.handleSubmit}
          >
            {/* Username Input */}
            <Input
              type="text"
              name="username"
              value={formik.values.username || ""}
              onChange={(e) => {
                const value = e.target.value || null;
                if (value && value.length > 30) return;
                formik.setFieldValue("username", value || null);
              }}
              onBlur={() => formik.setFieldTouched("username", true, false)}
              title={t("common.username")}
              icon={<User />}
              maxLength={30}
            />

            {/* Date Picker */}
            <CustomDatePicker
              value={formik.values.birthdayDate?.toString() || ""}
              onChange={(date) => formik.setFieldValue("birthdayDate", date)}
              title={t("common.birthday")}
              hasError={
                !!(formik.errors.birthdayDate && formik.touched.birthdayDate)
              }
              inputSize="large"
            />

            {/* Gender Selector */}
            <div className="grid grid-cols-3 gap-3">
              {Object.keys(GENDER).map((gender) => (
                <Button
                  key={gender}
                  className={`cursor-pointer py-4 px-2 border border-(--input-border) rounded-lg text-sm font-medium hover:border-primary transition-colors text-(--text-secondary) duration-900 ${formik.values.gender === gender ? "bg-(--primary-color) text-white" : ""}`}
                  type="button"
                  onClick={() => selectGender(gender as GENDER)}
                  title={t(`enum.${gender.toLowerCase()}`)}
                />
              ))}
            </div>

            {/* Footer Actions */}
            <div className="mt-4 flex gap-3">
              <Button
                type="button"
                onClick={onClose}
                disabled={LOADING_GOOGLE_SIGNUP}
                title={t("common.cancel")}
                className="flex-1 rounded-xl bg-(--bg-color) px-6 py-4 text-sm font-semibold text-(--text-primary) transition-all active:scale-[0.98] duration-100"
              />
              <Button
                disabled={isDisabled}
                isLoading={LOADING_GOOGLE_SIGNUP}
                title={t("common.complete_signup")}
                type="submit"
                className="flex-1 rounded-xl bg-(--primary-color) px-6 py-4 text-sm font-semibold text-[#050b08] shadow-[0_4px_20px_-5px_rgba(52,211,153,0.4)] transition-all duration-200 active:scale-[0.98]"
              />
            </div>
          </form>
        </div>
      )}
    </Modal>
  );
};

export default SignupModal;

```

## File: connectfy-client/src/modules/auth/ui/ForgotPassword/ForgotPassword.tsx
```typescript
import { useTranslation } from "react-i18next";
import { Fragment, useEffect } from "react";
import { ForgotPasswordModeType, IForgotPasswordForm } from "../../types/types";
import { useFormik } from "formik";
import { forgotPasswordInitialState } from "../../constants/intialState";
import { validateForgotPassword } from "../../constants/validation";
import { checkEmptyString } from "@/common/utils/checkValues";
import { useSearchParams } from "react-router-dom";
import { ROUTER } from "@/common/constants/routet";
import { snack } from "@/common/utils/snackManager";
import useFormDisabled from "@/hooks/useFormDisabled";
import Button from "@/components/ui/CustomButton/Button/Button";
import Input from "@/components/ui/CustomInput/Input/Input";
import PhoneNumber from "@/components/Form/PhoneNumberForm/PhoneNumberForm";
import { useForgotPasswordMutation } from "../../api/api";
import { useErrors } from "@/hooks/useErrors";
import { ShortcutTooltip } from "@/components/Tooltip/KeyboardShortcutTooltip";
import { ArrowLeft, Mail } from "lucide-react";
import { useAppNavigation } from "@/hooks/useAppNavigation";

const FORGOT_PASSWORD_MODES: ForgotPasswordModeType[] = [
  "email",
  "phoneNumber",
];

const ForgotPassword = () => {
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();
  const [searchParams, setSearchParams] = useSearchParams();

  const rawForgotPasswordMode = searchParams.get("mode");

  const forgotPasswordMode: ForgotPasswordModeType =
    FORGOT_PASSWORD_MODES.includes(
      rawForgotPasswordMode as ForgotPasswordModeType,
    )
      ? (rawForgotPasswordMode as ForgotPasswordModeType)
      : "email";

  const { showResponseErrors } = useErrors();

  const [forgotPassword, { isLoading: LOADING_FORGOT_PASSWORD }] =
    useForgotPasswordMutation();

  const formik = useFormik({
    initialValues: forgotPasswordInitialState,
    validateOnBlur: false,
    validateOnChange: false,
    enableReinitialize: true,
    validate: (values) => validateForgotPassword(values, t),
    onSubmit: async (values, { resetForm }) => {
      try {
        await forgotPassword(values).unwrap();
        snack.success(t("user_messages.forgot_password_successful"));
        navigate(ROUTER.AUTH.LOGIN);
        resetForm();
      } catch (error) {
        showResponseErrors(error);
      }
    },
  });

  const isDisabled = useFormDisabled<IForgotPasswordForm>({
    formik,
    loading: LOADING_FORGOT_PASSWORD,
    validationRules: [
      (values) => !!values.identifier && checkEmptyString(values.identifier),
    ],
  });

  const changeForgotPasswordMode = (mode: ForgotPasswordModeType) => {
    setSearchParams({ mode }, { replace: true });
  };

  useEffect(() => {
    const handleSubmitKeyboard = (e: KeyboardEvent) => {
      if (e.key === "Enter" && !isDisabled) {
        e.preventDefault();
        formik.handleSubmit();
      } else if (e.key === "Escape") {
        e.preventDefault();
        navigate(ROUTER.AUTH.LOGIN);
      } else if (e.shiftKey && e.altKey && e.code === "Digit1") {
        e.preventDefault();
        changeForgotPasswordMode("email");
      } else if (e.shiftKey && e.altKey && e.code === "Digit2") {
        e.preventDefault();
        changeForgotPasswordMode("phoneNumber");
      }
    };

    document.addEventListener("keydown", handleSubmitKeyboard);
    return () => {
      document.removeEventListener("keydown", handleSubmitKeyboard);
    };
  }, [isDisabled, formik.handleSubmit]);

  return (
    <Fragment>
      {/* Form Header */}
      <div className="space-y-2 mb-5">
        <h2 className="text-3xl font-bold tracking-tight text-(--text-(--primary-color))">
          {t("common.forgot_password_title")}
        </h2>
        <p className="text-(--text-secondary) text-sm">
          {t("common.forgot_password_subtitle")}
        </p>
      </div>

      {/* Login Mode Tabs */}
      <div className="flex border-b border-slate-100 my-4">
        <ShortcutTooltip keys={["Shift", "Alt", "1"]}>
          <Button
            type="button"
            className={`cursor-pointer px-4 py-2 w-full text-sm font-medium border-b-2 transition-colors ${
              forgotPasswordMode === "email"
                ? "text-(--primary-color) border-primary"
                : "text-slate-400 hover:text-slate-600 border-transparent"
            }`}
            onClick={() => changeForgotPasswordMode("email")}
          >
            {t("common.email")}
          </Button>
        </ShortcutTooltip>
        <ShortcutTooltip keys={["Shift", "Alt", "2"]}>
          <Button
            type="button"
            className={`cursor-pointer px-4 py-2 w-full text-sm font-medium border-b-2 transition-colors ${
              forgotPasswordMode === "phoneNumber"
                ? "text-(--primary-color) border-primary"
                : "text-slate-400 hover:text-slate-600 border-transparent"
            }`}
            onClick={() => changeForgotPasswordMode("phoneNumber")}
          >
            {t("common.phoneNumber")}
          </Button>
        </ShortcutTooltip>
      </div>

      {/* Login Form */}
      <form className="space-y-6" onSubmit={formik.handleSubmit}>
        {/* Identifier Input */}
        <div className="space-y-2">
          {forgotPasswordMode === "email" && (
            <Input
              className="w-full px-5 py-4 rounded-xl text-(--text-(--primary-color)) outline-none transition-all duration-200 placeholder:text-(--text-secondary)/50 focus:ring-2 focus:ring-[#34d399]/50"
              style={{
                backgroundColor: "var(--input-bg)",
                border: "1px solid var(--input-border)",
              }}
              title={t("common.email")}
              type="text"
              isFloating
              icon={<Mail size={20} />}
              name="identifier"
              onChange={formik.handleChange}
              onBlur={formik.handleBlur}
              value={formik.values.identifier || ""}
              isError={
                !!(formik.touched.identifier && formik.errors.identifier)
              }
              error={formik.errors.identifier}
              maxLength={254}
            />
          )}

          {forgotPasswordMode === "phoneNumber" && (
            <PhoneNumber
              name="phoneNumber"
              onChange={(value) =>
                formik.setFieldValue("identifier", value?.fullPhoneNumber)
              }
            />
          )}
        </div>

        {/* Submit Button */}
        <div className="flex flex-col items-center gap-4">
          <Button
            className="duration-400 h-[60px] w-full py-4 font-bold text-lg rounded-xl transition-all transform active:scale-[0.98] hover:brightness-110 shadow-[0_4px_14px_0_rgba(52,211,153,0.39)] bg-(--primary-color) text-[#020a06]"
            type="submit"
            disabled={isDisabled}
            title={t("common.login")}
            isLoading={LOADING_FORGOT_PASSWORD}
          />
          <p
            className="flex items-center gap-1 text-sm text-slate-500 dark:text-slate-400 hover:text-primary transition-colors mt-2 cursor-pointer"
            onClick={() => navigate(-1)}
          >
            <ArrowLeft size={20} className="md:text-md text-lg" />
            {t("common.back")}
          </p>
        </div>
      </form>
    </Fragment>
  );
};

export default ForgotPassword;

```

## File: connectfy-client/src/modules/auth/router/router.tsx
```typescript
import { RouteObject } from "react-router-dom";
import { lazy } from "react";
import { ROUTER } from "@/common/constants/routet";

const Main = lazy(() => import("../ui/Main/Main"));
const Login = lazy(() => import("../ui/Login/Login"));
const Signup = lazy(() => import("../ui/Signup/Signup"));
const VerifyLogin = lazy(() => import("../ui/VerifyLogin/VerifyLogin"));
const VerifySignup = lazy(() => import("../ui/VerifySignup/VerifySignup"));
const ForgotPassword = lazy(
  () => import("../ui/ForgotPassword/ForgotPassword"),
);
const ResetPassword = lazy(() => import("../ui/ResesPassword/ResetPassword"));

const routes: RouteObject[] = [
  {
    path: ROUTER.AUTH.MAIN,
    element: <Main />,
  },
  {
    path: ROUTER.AUTH.LOGIN,
    element: <Login />,
  },
  {
    path: ROUTER.AUTH.SIGNUP,
    element: <Signup />,
  },
  {
    path: ROUTER.AUTH.VERIFY_LOGIN,
    element: <VerifyLogin />,
  },
  {
    path: ROUTER.AUTH.VERIFY_SIGNUP,
    element: <VerifySignup />,
  },
  {
    path: ROUTER.AUTH.FORGOT_PASSWORD,
    element: <ForgotPassword />,
  },
  {
    path: ROUTER.AUTH.RESET_PASSWORD,
    element: <ResetPassword />,
  },
];

export default routes;

```

## File: connectfy-client/src/modules/auth/types/types.ts
```typescript
import {
  CHECK_UNIQUE_FIELD,
  FORGOT_PASSWORD_IDENTIFIER_TYPE,
  GENDER,
  IDENTIFIER_TYPE,
  LANGUAGE,
  STARTUP_PAGE,
  THEME,
  TOKEN_TYPE,
} from "@/common/enums/enums";

export type AuthFormType = "login" | "signup";

export type LoginModeType = "email" | "username" | "phoneNumber";

export type ForgotPasswordModeType = "email" | "phoneNumber";

export interface ILoginResponse {
  access_token: string;
  language: LANGUAGE;
  theme: THEME;
  startupPage: STARTUP_PAGE;
  isTwoFactorEnabled?: boolean;
}

export interface ISignupResponse {
  statusCode: number;
}

export interface ISignupVerifyResponse {
  _id: string;
  access_token: string;
}

export interface IForgotPasswordResponse {
  statusCode: number;
  email?: string;
}

export interface IResetPasswordResponse {
  statusCode: number;
}

export interface ILoginForm {
  identifierType: IDENTIFIER_TYPE;
  identifier: string | null;
  password: string | null;
}

export interface IPhoneNumber {
  countryCode: string | null;
  number: string | null;
  fullPhoneNumber: string | null;
}

export interface ISignupForm {
  firstName: string | null;
  lastName: string | null;
  username: string | null;
  email: string | null;
  gender: GENDER | null;
  password: string | null;
  confirm: string | null;
  birthdayDate: Date | null;
  theme: THEME;
  // phoneNumber: IPhoneNumber;
}

export interface ISignupVerifyForm {
  verifyCode: string | null;
}

export interface IForgotPasswordForm {
  identifierType: FORGOT_PASSWORD_IDENTIFIER_TYPE;
  identifier: string | null;
}

export interface IResetPasswordForm {
  password: string | null;
  confirmPassword: string | null;
  resetToken: string | null;
}

export interface IGoogleLoginForm {
  idToken: string | null;
}

export interface IGoogleSignupForm {
  idToken: string | null;
  username: string | null;
  // phoneNumber: IPhoneNumber;
  gender: GENDER | null;
  birthdayDate: Date | null;
  theme: THEME;
}

export interface IIsValidToken {
  token: string | null;
  type: TOKEN_TYPE;
}

export interface IRefreshResponse {
  user_id: string;
  access_token: string;
  refresh_token: string;
}

export interface IAuthenticateUser {
  password: string | null;
  type: TOKEN_TYPE;
  idToken: string | null;
}

export interface IAuthenticateUserResponse {
  statusCode: number;
  token: string;
}

export interface IRestoreAccount {
  token: string | null;
}

export interface IRestoreAccountResponse {
  access_token: string;
  language: LANGUAGE;
  theme: THEME;
  startupPage: STARTUP_PAGE;
}

export interface ICheckUnique {
  field: CHECK_UNIQUE_FIELD;
  value: string;
}

export interface ILoginVerifyForm {
  code: string | null;
}

export interface ILoginVerifyResponse extends ILoginResponse {}

```

## File: connectfy-client/src/modules/auth/api/api.ts
```typescript
import { baseQuery } from "@/common/api/axiosBaseQuery";
import { RESOURCE } from "@/common/enums/enums";
import { createApi } from "@reduxjs/toolkit/query/react";
import {
  ICheckUnique,
  ILoginForm,
  ILoginResponse,
  ISignupForm,
  ISignupResponse,
  ISignupVerifyForm,
  ISignupVerifyResponse,
  IForgotPasswordResponse,
  IForgotPasswordForm,
  IResetPasswordResponse,
  IResetPasswordForm,
  IGoogleLoginForm,
  IGoogleSignupForm,
  IIsValidToken,
  IRefreshResponse,
  IAuthenticateUserResponse,
  IRestoreAccountResponse,
  IRestoreAccount,
  IAuthenticateUser,
  ILoginVerifyResponse,
  ILoginVerifyForm,
} from "../types/types";
import { API_ENDPOINTS } from "@/common/constants/apiEndpoints";

export const authApi = createApi({
  reducerPath: RESOURCE.AUTH,
  baseQuery: baseQuery,
  tagTypes: ["Auth"],
  endpoints: (builder) => ({
    // ====================== LOGIN
    login: builder.mutation<ILoginResponse, ILoginForm>({
      query: (data) => ({
        url: API_ENDPOINTS.AUTH.LOGIN,
        method: "POST",
        body: data,
      }),
      invalidatesTags: ["Auth"],
    }),

    // ====================== VERIFY LOGIN
    loginVerify: builder.mutation<ILoginVerifyResponse, ILoginVerifyForm>({
      query: (data) => ({
        url: API_ENDPOINTS.AUTH.LOGIN_VERIFY,
        method: "POST",
        body: data,
      }),
      invalidatesTags: ["Auth"],
    }),

    // ====================== SINGUP
    signup: builder.mutation<ISignupResponse, Omit<ISignupForm, "confirm">>({
      query: (data) => ({
        url: API_ENDPOINTS.AUTH.SIGNUP,
        method: "POST",
        body: data,
      }),
      invalidatesTags: ["Auth"],
    }),

    // ====================== VERIFY SIGNUP
    signupVerify: builder.mutation<ISignupVerifyResponse, ISignupVerifyForm>({
      query: (data) => ({
        url: API_ENDPOINTS.AUTH.SIGNUP_VERIFY,
        method: "POST",
        body: data,
      }),
      invalidatesTags: ["Auth"],
    }),

    // ======================== RESEND SIGNUP VERIFY
    resendSignupVerify: builder.mutation<ISignupVerifyResponse, void>({
      query: () => ({
        url: API_ENDPOINTS.AUTH.RESEND_SIGNUP_VERIFY,
        method: "POST",
      }),
      invalidatesTags: ["Auth"],
    }),

    // ======================== CHECK UNIQUE
    checkUnique: builder.mutation<boolean, ICheckUnique>({
      query: (data) => ({
        url: API_ENDPOINTS.USER.CHECK_UNIQUE,
        method: "POST",
        body: data,
      }),
      invalidatesTags: ["Auth"],
    }),

    // ======================== GOOGLE LOGIN
    googleLogin: builder.mutation<ILoginResponse, IGoogleLoginForm>({
      query: (data) => ({
        url: API_ENDPOINTS.AUTH.GOOGLE_LOGIN,
        method: "POST",
        body: data,
      }),
      invalidatesTags: ["Auth"],
    }),

    // ======================== GOOGLE SIGNUP
    googleSignup: builder.mutation<ISignupVerifyResponse, IGoogleSignupForm>({
      query: (data) => ({
        url: API_ENDPOINTS.AUTH.GOOGLE_SIGNUP,
        method: "POST",
        body: data,
      }),
      invalidatesTags: ["Auth"],
    }),

    // ======================== FORGOT PASSWORD
    forgotPassword: builder.mutation<
      IForgotPasswordResponse,
      IForgotPasswordForm
    >({
      query: (data) => ({
        url: API_ENDPOINTS.AUTH.FORGOT_PASSWORD,
        method: "POST",
        body: data,
      }),
      invalidatesTags: ["Auth"],
    }),

    // ======================== RESET PASSWORD
    resetPassword: builder.mutation<IResetPasswordResponse, IResetPasswordForm>(
      {
        query: (data) => ({
          url: API_ENDPOINTS.AUTH.RESET_PASSWORD,
          method: "POST",
          body: data,
        }),
        invalidatesTags: ["Auth"],
      },
    ),

    // ======================== VALIDATE TOKEN
    isValidToken: builder.mutation<boolean, IIsValidToken>({
      query: (data) => ({
        url: API_ENDPOINTS.AUTH.IS_VALID_TOKEN,
        method: "POST",
        body: data,
      }),
      invalidatesTags: ["Auth"],
    }),

    // ======================== REFRESH
    refresh: builder.mutation<IRefreshResponse, void>({
      query: () => ({
        url: API_ENDPOINTS.AUTH.REFRESH,
        method: "POST",
      }),
      invalidatesTags: ["Auth"],
    }),

    // ======================== AUTHENTICATE USER
    authenticateUser: builder.mutation<
      IAuthenticateUserResponse,
      IAuthenticateUser
    >({
      query: (data) => ({
        url: API_ENDPOINTS.AUTH.AUTHENTICATE_USER,
        method: "POST",
        body: data,
      }),
      invalidatesTags: ["Auth"],
    }),

    // ======================== RESTORE ACCOUNT
    restoreAccount: builder.mutation<IRestoreAccountResponse, IRestoreAccount>({
      query: (data) => ({
        url: API_ENDPOINTS.AUTH.RESTORE_ACCOUNT,
        method: "POST",
        body: data,
      }),
      invalidatesTags: ["Auth"],
    }),
  }),
});

export const {
  useLoginMutation,
  useLoginVerifyMutation,
  useSignupMutation,
  useSignupVerifyMutation,
  useResendSignupVerifyMutation,
  useCheckUniqueMutation,
  useGoogleLoginMutation,
  useGoogleSignupMutation,
  useForgotPasswordMutation,
  useResetPasswordMutation,
  useIsValidTokenMutation,
  useRefreshMutation,
  useAuthenticateUserMutation,
  useRestoreAccountMutation,
} = authApi;

```

## File: connectfy-client/src/modules/messenger/ui/Messenger.tsx
```typescript
import { FC } from "react";

const Messenger: FC = () => {
  return <div>MessengerPage</div>;
};

export default Messenger;

```

## File: connectfy-client/src/modules/termsAndConditions/ui/TermsAndConditions.tsx
```typescript
import "./termsAndConditions.style.css";
import { useCallback, useState, Fragment } from "react";
import { useTranslation } from "react-i18next";
import { LANGUAGE } from "@/common/enums/enums";
import { snack } from "@/common/utils/snackManager";
import SelectionModal from "@/components/Modal/SelectionModal/SelectionModal";
import Input from "@/components/ui/CustomInput/Input/Input";
import Button from "@/components/ui/CustomButton/Button/Button";
import { Globe, ClipboardList, ClipboardCheck } from "lucide-react";
import TextTooltip from "@/components/Tooltip/TextTooltip";

const TermsAndConditions = () => {
  const { i18n, t } = useTranslation();

  const lang = i18n.language;

  const [openLangModal, setOpenLangModal] = useState<boolean>(false);

  const languageList = [
    {
      name: "English",
      value: LANGUAGE.EN,
      icon: Globe,
      key: "EN",
      onClick: () => i18n.changeLanguage(LANGUAGE.EN),
    },
    {
      name: "Azərbaycan",
      value: LANGUAGE.AZ,
      icon: Globe,
      key: "AZ",
      onClick: () => i18n.changeLanguage(LANGUAGE.AZ),
    },
    {
      name: "Русский",
      value: LANGUAGE.RU,
      icon: Globe,
      key: "RU",
      onClick: () => i18n.changeLanguage(LANGUAGE.RU),
    },
    {
      name: "Türkçe",
      value: LANGUAGE.TR,
      icon: Globe,
      key: "TR",
      onClick: () => i18n.changeLanguage(LANGUAGE.TR),
    },
  ];

  const renderLangIcon = useCallback(() => {
    switch (lang) {
      case LANGUAGE.AZ:
        return <span className="fi fi-az"></span>;
      case LANGUAGE.TR:
        return <span className="fi fi-tr"></span>;
      case LANGUAGE.RU:
        return <span className="fi fi-ru"></span>;
      default:
        return <span className="fi fi-gb"></span>;
    }
  }, [lang]);

  const [copied, setCopied] = useState(false);
  const email = "connectfy.team@gmail.com";

  const handleCopy = () => {
    window.navigator.clipboard.writeText(email);
    setCopied(true);

    if (typeof snack !== "undefined") {
      snack.success(t("common.email_copied"));
    }

    setTimeout(() => setCopied(false), 2000);
  };

  return (
    <Fragment>
      <TextTooltip position="top" text={t("common.change_lang")}>
        <div className="lang-switcher" onClick={() => setOpenLangModal(true)}>
          {renderLangIcon()}
        </div>
      </TextTooltip>

      <main className="terms-container">
        <h1 className="terms-title">{t("terms.main_title")}</h1>

        <section className="terms-section">
          <h2>{t("terms.section_1_title")}</h2>
          <p>{t("terms.section_1_p1")}</p>
        </section>

        <section className="terms-section">
          <h2>{t("terms.section_2_title")}</h2>
          <p>{t("terms.section_2_p1")}</p>
          <p>{t("terms.section_2_p2")}</p>
        </section>

        <section className="terms-section">
          <h2>{t("terms.section_3_title")}</h2>
          <p>{t("terms.section_3_p1")}</p>
          <p>{t("terms.section_3_p2")}</p>
        </section>

        <section className="terms-section">
          <h2>{t("terms.section_4_title")}</h2>
          <p>{t("terms.section_4_p1")}</p>
          <p>{t("terms.section_4_p2")}</p>
        </section>

        <section className="terms-section">
          <h2>{t("terms.section_5_title")}</h2>
          <p>{t("terms.section_5_p1")}</p>
          <ul>
            <li>{t("terms.section_5_list_1")}</li>
            <li>{t("terms.section_5_list_2")}</li>
            <li>{t("terms.section_5_list_3")}</li>
          </ul>
          <p>{t("terms.section_5_p2")} </p>
        </section>

        <section className="terms-section">
          <h2>{t("terms.section_6_title")}</h2>
          <p>{t("terms.section_6_p1")}</p>
        </section>

        <section className="terms-section">
          <h2>{t("terms.section_7_title")}</h2>
          <p>{t("terms.section_7_p1")}</p>
          <p>{t("terms.section_7_p2")}</p>
        </section>

        <section className="terms-section">
          <h2>{t("terms.section_8_title")}</h2>
          <p>{t("terms.section_8_p1")}</p>
        </section>

        <section className="terms-section">
          <h2>{t("terms.section_9_title")}</h2>
          <p>{t("terms.section_9_p1")}</p>
        </section>

        <section className="terms-section">
          <h2>{t("terms.section_10_title")}</h2>
          <p>{t("terms.section_10_p1")}</p>
        </section>

        <section className="terms-section">
          <h2>{t("terms.section_11_title")}</h2>
          <p>{t("terms.section_11_p1")}</p>
          <div className="">
            <div className="relative">
              <label htmlFor="email-copy-text" className="sr-only">
                Email
              </label>
              <Input
                inputSize="medium"
                id="email-copy-text"
                type="text"
                className="col-span-6 border text-sm rounded-lg block w-full px-3 py-2.5 shadow-sm focus:outline-none bg-(--input-bg) border-(--input-border) text-(--text-primary)"
                value={email}
                // disabled
                readOnly
              />
              <Button
                onClick={handleCopy}
                className="absolute flex items-center end-1.5 top-1/2 -translate-y-1/2 border font-medium leading-5 rounded text-xs px-3 py-1.5 focus:outline-none transition-colors bg-(--primary-color) text-(--text-primary) border-(--auth-glass-border)"
              >
                {!copied ? (
                  <ClipboardList color={"white"} />
                ) : (
                  <ClipboardCheck color={"white"} />
                )}
              </Button>
            </div>
          </div>
        </section>

        <footer className="terms-footer">
          <p>{t("terms.footer_text")}</p>
        </footer>
      </main>

      <SelectionModal
        open={openLangModal}
        onClose={() => setOpenLangModal(false)}
        title={t("common.change_lang")}
        selections={languageList}
        activeKey={(lang as string).toUpperCase()}
      />
    </Fragment>
  );
};

export default TermsAndConditions;

```

## File: connectfy-client/src/modules/termsAndConditions/router/router.tsx
```typescript
import { RouteObject } from "react-router-dom";
import { lazy } from "react";
import { ROUTER } from "@/common/constants/routet";
import Loader from "@/components/Loader/Main/Loader.tsx";

const TermsAndConditions = Loader(lazy(() => import("../ui/TermsAndConditions")));

const routes: RouteObject[] = [
  {
    path: ROUTER.TERMS_AND_CONDITIONS,
    element: <TermsAndConditions />,
  },
];

export default routes;

```

## File: connectfy-client/src/modules/users/UserProfile/ui/Profile.tsx
```typescript
import { FC, Fragment, memo } from "react";
import ProfileHeader from "./components/ProfileHeader/ProfileHeader";
import MainCard from "./components/MainCard/MainCard";
import PersonalInformation from "./components/PersonalInformation/PersonalInformation";
import Bio from "./components/Bio/Bio";
import SocialLinks from "./components/SocialLinks/SocialLinks";
import MainCardSkeleton from "@/components/Skeleton/profile/MainCardSkeleton";
import PersonalInformationSkeleton from "@/components/Skeleton/profile/PersonalInformationSkeleton";
import BioSkeleton from "@/components/Skeleton/profile/BioSkeleton";
import { useFindUserQuery } from "../api/api";
import { Navigate, useParams } from "react-router-dom";
import { validate } from "uuid";

const Profile: FC = () => {
  const { id } = useParams<{ id: string }>();

  // UUID yoxlamasını bir dəyişəndə saxlayırıq
  const isValidId = id && validate(id);

  // 1. Hook mütləq yuxarıda olmalıdır. Əgər ID səhvdirsə, `skip` ilə API sorğusunu dayandırırıq.
  const { data, isLoading, isError } = useFindUserQuery(id as string, {
    skip: !isValidId,
  });

  // 2. Early Return-ləri Hook-dan sonra edirik
  if (!isValidId) {
    return <Navigate to="/" replace />;
  }

  if (!isLoading && (!data || isError)) {
    return <Navigate to="/" replace />;
  }

  return (
    <div className="flex w-full h-screen overflow-hidden bg-(--bg-color) font-sans">
      <div className="flex-1 h-full overflow-x-hidden overflow-y-auto scroll-smooth">
        <ProfileHeader relationship={data?.relationship} user={data?.user} actions={data?.actions} />

        <main className="mx-auto max-w-[900px] pt-8 px-6 pb-[60px] animate-slide-up">
          {/* 3. Data yoxdursa (yəni yüklənirsə), Skeleton-ları göstəririk. Destructuring xətası olmur. */}
          {isLoading || !data ? (
            <Fragment>
              <MainCardSkeleton />
              <PersonalInformationSkeleton />
              <BioSkeleton />
            </Fragment>
          ) : data.actions?.isBlocked ? (
            <div className="flex flex-col items-center justify-center pt-20">
              <h2 className="text-xl font-semibold text-(--text-color)">
                User not found
              </h2>
            </div>
          ) : (
            <Fragment>
              {/* 4. Data yalnız bu blokda mövcud (defined) olduğu üçün təhlükəsiz ötürürük */}
              <MainCard
                user={data.user}
                profile={data.profile}
                actions={data.actions}
                relationship={data.relationship}
              />
              <PersonalInformation user={data.user} profile={data.profile} />
              <Bio profile={data.profile} />

              <SocialLinks userId={data.user._id} />
            </Fragment>
          )}
        </main>
      </div>
    </div>
  );
};

export default memo(Profile);

```

## File: connectfy-client/src/modules/users/UserProfile/ui/components/Bio/Bio.tsx
```typescript
import { FC, Fragment } from "react";
import { UserPen, MinusIcon } from "lucide-react";
import { useTranslation } from "react-i18next";
import { IFindOneProfile } from "../../../types/types";

interface IProps {
  profile: IFindOneProfile;
}

const Bio: FC<IProps> = ({ profile }) => {
  const { t } = useTranslation();

  return (
    <Fragment>
      <section
        className="p-8 mb-8 transition-all duration-300 bg-(--auth-main-bg) rounded-[20px] border border-(--auth-glass-border) shadow-(--card-shadow)"
        aria-labelledby="bio-heading"
      >
        {/* Header Hissəsi */}
        <div className="flex flex-row items-center justify-between gap-3 mb-6 md:mb-8">
          {/* Sol tərəf: İkon və Başlıq */}
          <div className="flex items-center gap-2 md:gap-3">
            <UserPen className="w-6 h-6 md:w-7 md:h-7 text-(--primary-color)" />
            <h2
              id="bio-heading"
              className="m-0 text-lg font-bold md:text-2xl text-(--text-color)"
            >
              {t("common.bio")}
            </h2>
          </div>
        </div>

        {/* Bio Mətni */}

        <div className="p-6 rounded-xl bg-(--info-card-bg) border border-(--info-card-border) transition-all duration-200 hover:bg-(--active-bg-2)">
          {profile?.bio ? (
            <div
              className="text-[15px] leading-[1.7] text-(--text-color) prose prose-sm max-w-none [&_strong]:font-bold [&_em]:italic [&_ul]:list-disc [&_ul]:pl-5 [&_ol]:list-decimal [&_ol]:pl-5 [&_s]:line-through"
              dangerouslySetInnerHTML={{ __html: profile.bio }}
            />
          ) : (
            <div className="flex items-center gap-2 text-(--muted-color) italic py-2">
              <MinusIcon size={18} />
              <span>{t("common.no_bio_info")}</span>
            </div>
          )}
        </div>
      </section>
    </Fragment>
  );
};

export default Bio;

```

## File: connectfy-client/src/modules/users/UserProfile/ui/components/Modal/AvatarModal/AvatarModal.tsx
```typescript
import { FC, useState } from "react";
import { motion, AnimatePresence } from "framer-motion";
import { X, Copy, ZoomIn, ZoomOut, QrCode, ArrowLeft } from "lucide-react";
import { QRCodeCanvas } from "qrcode.react";
import Modal from "@/components/Modal";
import { useTranslation } from "react-i18next";
import Button from "@/components/ui/CustomButton/Button/Button";
import { snack } from "@/common/utils/snackManager";
import TextTooltip from "@/components/Tooltip/TextTooltip";
import SharePopover from "@/components/Modal/AvatarModal/SharePopover/SharePopover";

interface IProps {
  open: boolean;
  onClose: () => void;
  avatarUrl: string;
  username?: string;
  userId?: string;
}

const AvatarModal: FC<IProps> = ({
  open,
  onClose,
  avatarUrl,
  username,
  userId,
}) => {
  const { t } = useTranslation();

  const [scale, setScale] = useState(1);
  const [showQr, setShowQr] = useState(false);

  const profileUrl = `${window.location.origin}/users/profile/${userId}`;

  const handleZoomIn = () => setScale((prev) => Math.min(prev + 0.5, 3));
  const handleZoomOut = () => setScale((prev) => Math.max(prev - 0.5, 1));

  const handleCopyLink = () => {
    navigator.clipboard.writeText(profileUrl);
    snack.success(t("common.link_copied"));
  };

  const handleToggleQr = () => {
    setShowQr((prev) => !prev);
    setScale(1);
  };

  return (
    <Modal open={open} onClose={onClose}>
      <motion.div
        initial={{ opacity: 0, scale: 0.9 }}
        animate={{ opacity: 1, scale: 1 }}
        exit={{ opacity: 0, scale: 0.9 }}
        className="relative flex flex-col w-full max-w-lg overflow-hidden bg-(--auth-main-bg) rounded-3xl shadow-2xl"
      >
        {/* Header */}
        <div className="absolute top-0 left-0 right-0 z-20 flex items-center justify-between p-4 bg-linear-to-b from-black/40 to-transparent">
          <div className="flex items-center gap-2">
            {showQr && (
              <Button
                onClick={handleToggleQr}
                className="p-1.5 bg-white/10 hover:bg-white/20 backdrop-blur-md rounded-full text-white transition-all"
              >
                <ArrowLeft size={18} />
              </Button>
            )}
            <span className="font-medium text-white drop-shadow-md">
              {showQr
                ? t("common.qr_code")
                : username || t("common.profile_photo")}
            </span>
          </div>
          <Button
            onClick={onClose}
            className="p-2 transition-transform bg-white/10 hover:bg-white/20 backdrop-blur-md rounded-full text-white hover:scale-110"
          >
            <X size={20} />
          </Button>
        </div>

        {/* Image / QR Toggle Area */}
        <div className="relative flex items-center justify-center w-full aspect-square overflow-hidden group">
          <AnimatePresence mode="wait">
            {showQr ? (
              // ── QR Görünüşü ──
              <motion.div
                key="qr"
                initial={{ opacity: 0, scale: 0.85, rotateY: 90 }}
                animate={{ opacity: 1, scale: 1, rotateY: 0 }}
                exit={{ opacity: 0, scale: 0.85, rotateY: -90 }}
                transition={{ type: "spring", stiffness: 180, damping: 22 }}
                className="flex flex-col items-center justify-center gap-5 w-full h-full bg-(--auth-main-bg)"
              >
                {/* QR Wrapper - Instagram kimi ağ fon + avatar */}
                <div className="relative p-5 bg-white rounded-3xl shadow-xl">
                  <QRCodeCanvas
                    id="avatar-qr-code"
                    value={profileUrl}
                    size={200}
                    bgColor="#ffffff"
                    fgColor="#000000"
                    level="H"
                    imageSettings={{
                      src: avatarUrl,
                      x: undefined,
                      y: undefined,
                      height: 48,
                      width: 48,
                      excavate: true,
                    }}
                  />
                </div>

                {/* @username */}
                <div className="flex flex-col items-center gap-1">
                  <span className="text-lg font-bold text-(--text-primary)">
                    @{username}
                  </span>
                  <span className="text-xs text-(--text-secondary) opacity-60">
                    {t("common.scan_to_view_profile")}
                  </span>
                </div>
              </motion.div>
            ) : (
              // ── Şəkil Görünüşü ──
              <motion.div
                key="image"
                initial={{ opacity: 0 }}
                animate={{ opacity: 1 }}
                exit={{ opacity: 0 }}
                className="relative w-full h-full"
              >
                <motion.img
                  animate={{ scale }}
                  transition={{ type: "spring", stiffness: 200, damping: 25 }}
                  src={avatarUrl}
                  alt="Avatar"
                  className="object-contain w-full h-full"
                  onDoubleClick={scale > 1 ? () => setScale(1) : handleZoomIn}
                />
                {/* Zoom Controls */}
                <div className="absolute bottom-4 right-4 flex flex-col gap-2 opacity-0 group-hover:opacity-100 transition-opacity duration-300">
                  <Button
                    onClick={handleZoomIn}
                    className="p-2.5 bg-white/10 backdrop-blur-md border border-white/20 rounded-xl text-white hover:bg-white/20"
                  >
                    <ZoomIn size={18} />
                  </Button>
                  <Button
                    onClick={handleZoomOut}
                    className="p-2.5 bg-white/10 backdrop-blur-md border border-white/20 rounded-xl text-white hover:bg-white/20"
                  >
                    <ZoomOut size={18} />
                  </Button>
                </div>
              </motion.div>
            )}
          </AnimatePresence>
        </div>

        {/* Action Bar */}
        <div className="flex items-center justify-around p-5 bg-(--auth-main-bg) border-t border-(--auth-glass-border)">
          {/* 1. Share (Paylaş) */}
          <SharePopover
            profileUrl={profileUrl}
            username={username}
            text={t("common.profile_text", { username })}
          />

          {/* 2. Copy Link (Linki kopyala) */}
          <TextTooltip position="top" text={t("common.copy_link")}>
            <Button
              onClick={handleCopyLink}
              className="w-[90px] flex flex-col items-center gap-2 transition-colors text-(--text-secondary) hover:text-(--third-color) group min-w-0"
            >
              <div className="p-3 rounded-2xl bg-(--active-bg-2) group-hover:bg-(--active-bg) transition-colors shrink-0">
                <Copy size={22} />
              </div>
            </Button>
          </TextTooltip>

          {/* 3. QR Code (QR Kod) */}
          <TextTooltip position="top" text={t("common.qr_code")}>
            <Button
              onClick={handleToggleQr}
              className={`w-[90px] flex flex-col items-center gap-2 transition-colors group min-w-0 ${
                showQr
                  ? "text-(--third-color)"
                  : "text-(--text-secondary) hover:text-(--third-color)"
              }`}
            >
              <div
                className={`p-3 rounded-2xl transition-colors shrink-0 ${
                  showQr
                    ? "bg-(--active-bg)"
                    : "bg-(--active-bg-2) group-hover:bg-(--active-bg)"
                }`}
              >
                <QrCode size={22} />
              </div>
            </Button>
          </TextTooltip>
        </div>
      </motion.div>
    </Modal>
  );
};

export default AvatarModal;

```

## File: connectfy-client/src/modules/users/UserProfile/ui/components/Modal/MutualFriendsModal/MutualFriendsModal.tsx
```typescript
import { FC, useState } from "react";
import { motion } from "framer-motion";
import { X } from "lucide-react";
import Modal from "@/components/Modal";
import { useTranslation } from "react-i18next";
import Button from "@/components/ui/CustomButton/Button/Button";
import { useFindMutualFriendsQuery } from "@/modules/users/MyFriends/api/api";
import UserCard from "@/components/Card/UserCard/UserCard";
import { LoadingSpinner } from "@/components/Spinner/Settings/LoadingSpinner";
import { useAppNavigation } from "@/hooks/useAppNavigation";
import { ROUTER } from "@/common/constants/routet";

interface IProps {
  open: boolean;
  onClose: () => void;
  targetUserId: string;
}

const MutualFriendsModal: FC<IProps> = ({ open, onClose, targetUserId }) => {
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();
  const [page, setPage] = useState(1);
  const limit = 10;

  const { data, isLoading, isFetching } = useFindMutualFriendsQuery(
    { targetUserId, skip: page, limit },
    { skip: !open },
  );

  const friends = data?.data || [];
  const count = data?.count || 0;
  const totalPages = Math.ceil(count / limit);

  return (
    <Modal open={open} onClose={onClose}>
      <motion.div
        initial={{ opacity: 0, scale: 0.9 }}
        animate={{ opacity: 1, scale: 1 }}
        exit={{ opacity: 0, scale: 0.9 }}
        className="relative flex flex-col w-full max-w-lg bg-(--auth-main-bg) rounded-3xl shadow-2xl overflow-hidden"
      >
        <div className="flex items-center justify-between p-5 border-b border-(--shadow-color)">
          <h2 className="text-lg font-bold text-(--text-color)">
            {t("common.mutual_friends", "Mutual Friends")} ({count})
          </h2>
          <Button
            onClick={onClose}
            icon={<X size={20} />}
            className="p-2 transition-colors rounded-full text-(--text-color) hover:bg-(--active-bg-2)"
          />
        </div>

        <div className="p-4 overflow-y-auto max-h-[60vh] space-y-2">
          {isLoading || isFetching ? (
            <div className="flex justify-center p-4">
              <LoadingSpinner />
            </div>
          ) : friends.length > 0 ? (
            friends.map((user: any) => (
              <UserCard
                key={user.userId}
                user={{
                  _id: user.userId,
                  firstName: user.firstName,
                  lastName: user.lastName,
                  fullName: user.fullName,
                  username: user.username,
                  avatar: {
                    key: "",
                    isCustom: false,
                    url: user.avatar,
                  },
                }}
                onClick={({ userId }) => {
                  navigate(`${ROUTER.USERS.PROFILE}/${userId}`);
                  onClose();
                }}
              />
            ))
          ) : (
            <p className="text-center text-(--muted-color) py-4">
              {t("common.no_mutual_friends", "No mutual friends found.")}
            </p>
          )}
        </div>

        {totalPages > 1 && (
          <div className="flex items-center justify-center gap-4 p-4 border-t border-(--shadow-color)">
            <Button
              disabled={page === 1}
              onClick={() => setPage((p) => Math.max(1, p - 1))}
              title={t("common.previous", "Previous")}
              className="px-4 py-2 text-sm font-medium rounded-lg bg-(--active-bg-2) disabled:opacity-50 text-(--text-color)"
            />
            <span className="text-sm text-(--text-color)">
              {page} / {totalPages}
            </span>
            <Button
              disabled={page === totalPages}
              onClick={() => setPage((p) => Math.min(totalPages, p + 1))}
              title={t("common.next", "Next")}
              className="px-4 py-2 text-sm font-medium rounded-lg bg-(--active-bg-2) disabled:opacity-50 text-(--text-color)"
            />
          </div>
        )}
      </motion.div>
    </Modal>
  );
};

export default MutualFriendsModal;

```

## File: connectfy-client/src/modules/users/UserProfile/ui/components/Modal/ShareModal/ShareModal.tsx
```typescript
import Modal from "@/components/Modal";
import { FC, Fragment, useState } from "react";
import { Check, Copy, ExternalLink } from "lucide-react";
import { useShare } from "@/hooks/useShare";
import Button from "@/components/ui/CustomButton/Button/Button";
import { useTranslation } from "react-i18next";

interface IProps {
  open: boolean;
  onClose: () => void;
  profileUrl: string;
  username?: string;
}

const ShareModal: FC<IProps> = ({ open, onClose, profileUrl, username }) => {
  const { t } = useTranslation();
  const [copied, setCopied] = useState(false);

  const { shareLinks, handleOpen } = useShare({
    url: profileUrl,
    text: username ? t("common.share_text", { username }) : "",
  });

  const handleCopy = async () => {
    try {
      await navigator.clipboard.writeText(profileUrl);
      setCopied(true);
      setTimeout(() => setCopied(false), 2000);
    } catch {
      // fallback
      const el = document.createElement("input");
      el.value = profileUrl;
      document.body.appendChild(el);
      el.select();
      document.execCommand("copy");
      document.body.removeChild(el);
      setCopied(true);
      setTimeout(() => setCopied(false), 2000);
    }
  };

  const handleNativeShare = async () => {
    if (navigator.share) {
      try {
        await navigator.share({ url: profileUrl, title: username });
      } catch {
        /* user cancelled */
      }
    }
  };

  return (
    <Modal open={open} onClose={onClose}>
      <div className="flex flex-col gap-5 p-5 bg-(--bg-color) rounded-xl w-full md:max-w-[500px] animate-slide-up">
        {/* Header */}
        <div>
          <h2 className="text-base font-semibold text-(--text-color) m-0">
            {t("common.share_profile")}
          </h2>
          {username && (
            <p className="text-sm text-(--muted-color) mt-0.5 mb-0">
              @{username}
            </p>
          )}
        </div>

        {/* URL copy row */}
        <div>
          <p className="text-xs text-(--muted-color) mb-2">
            {t("common.link")}
          </p>
          <div className="flex items-center gap-2 bg-(--input-bg) border border-(--input-border) rounded-xl px-3.5 py-2.5">
            <span className="flex-1 text-sm text-(--muted-color) truncate">
              {profileUrl}
            </span>
            <Button
              onClick={handleCopy}
              className={`
                flex items-center gap-1.5 text-xs font-medium px-3 py-1.5 rounded-lg
                border-none cursor-pointer transition-all duration-200 shrink-0
                ${
                  copied
                    ? "bg-(--active-bg) text-(--primary-color)"
                    : "bg-(--active-bg-2) text-(--primary-color) hover:bg-(--active-bg)"
                }
              `}
            >
              {copied ? (
                <Fragment>
                  <Check size={13} strokeWidth={2.5} />
                  {t("common.copied")}
                </Fragment>
              ) : (
                <Fragment>
                  <Copy size={13} strokeWidth={2} />
                  {t("common.copy")}
                </Fragment>
              )}
            </Button>
          </div>
        </div>

        {/* Social icons */}
        <div>
          <p className="text-xs text-(--muted-color) mb-3">Share via</p>
          <div className="grid grid-cols-4 gap-2">
            {shareLinks.map((item) => (
              <Button
                key={item.label}
                onClick={() => handleOpen(item.href)}
                className="
                  flex flex-col items-center gap-2
                  bg-transparent border-none cursor-pointer p-0
                  group
                "
              >
                <span
                  className="
                    w-13 h-13 rounded-2xl
                    flex items-center justify-center
                    bg-(--active-bg-2) border border-(--auth-glass-border)
                    transition-all duration-200
                    group-hover:bg-(--active-bg) group-hover:scale-105
                    group-active:scale-95
                  "
                >
                  {item.icon}
                </span>
                <span className="text-[11px] text-(--muted-color) group-hover:text-(--text-color) transition-colors">
                  {item.label}
                </span>
              </Button>
            ))}
          </div>
        </div>

        {/* Divider */}
        <div className="h-px bg-(--auth-glass-border)" />

        {/* Native share / bottom CTA */}
        <Button
          onClick={
            typeof navigator !== "undefined" && "share" in navigator
              ? handleNativeShare
              : handleCopy
          }
          className="
            w-full flex items-center justify-center gap-2
            py-3 rounded-xl border-none cursor-pointer
            text-sm font-medium text-(--primary-color)
            bg-(--active-bg-2)
            transition-all duration-200
            hover:bg-(--active-bg) hover:-translate-y-0.5
            active:translate-y-0
          "
          icon={<ExternalLink size={16} strokeWidth={2} />}
          title={
            typeof navigator !== "undefined" && "share" in navigator
              ? t("common.more_options")
              : t("common.copy_link")
          }
        />
      </div>
    </Modal>
  );
};

export default ShareModal;

```

## File: connectfy-client/src/modules/users/UserProfile/ui/components/PersonalInformation/PersonalInformation.tsx
```typescript
import {
  Cake,
  Phone,
  Mail,
  MapPin,
  MapMinusIcon,
  Contact,
  Users,
  PhoneOff,
  MailMinus,
  CalendarMinus,
  CircleOff,
} from "lucide-react";
import { FC, Fragment } from "react";
import { useTranslation } from "react-i18next";
import { DDMMMMYYYY, formatPhoneNumber } from "@/common/utils/formatValues";
import { COUNTRIES } from "@/common/constants/constants";
import { useIsMobile } from "@/hooks/useIsMobile";
import { IFindOneProfile, IFindOneUser } from "../../../types/types";

export interface IInfoItem {
  id: string;
  icon: React.ReactNode;
  label: string;
  value: string | React.ReactNode;
  colorClass: string;
}

interface IProps {
  user: IFindOneUser;
  profile: IFindOneProfile;
}

const PersonalInformation: FC<IProps> = ({ user, profile }) => {
  const { t } = useTranslation();
  const isMobile = useIsMobile();

  const phoneNumberMask = COUNTRIES.find(
    (country) => country.code === user?.phoneNumber?.countryCode,
  )?.format;

  // Məlumat kartlarını dinamik render etmək üçün obyektlər massivi
  const infoItems: IInfoItem[] = [
    {
      id: "email",
      icon: <Mail size={isMobile ? 15 : 20} />,
      label: t("common.email"),
      value: user?.email || (
        <span className="flex items-center gap-1 italic font-medium text-slate-400">
          <MailMinus size={16} /> {t("common.no_email_data")}
        </span>
      ),
      colorClass:
        "bg-indigo-100 text-indigo-600 dark:bg-indigo-500/15 dark:text-indigo-400",
    },
    {
      id: "gender",
      icon: <Users size={isMobile ? 15 : 20} />,
      label: t("common.gender"),
      value: profile?.gender ? (
        t(`enum.${profile.gender.toLowerCase()}`)
      ) : (
        <span className="flex items-center gap-1 italic font-medium text-slate-400">
          <CircleOff size={16} /> {t("common.no_gender_data")}
        </span>
      ),
      colorClass:
        "bg-purple-100 text-purple-600 dark:bg-purple-500/15 dark:text-purple-400",
    },
    {
      id: "phone",
      icon: <Phone size={isMobile ? 15 : 20} />,
      label: t("common.phoneNumber"),
      value: user?.phoneNumber ? (
        formatPhoneNumber(
          user?.phoneNumber?.number as string,
          phoneNumberMask as string,
          user?.phoneNumber?.countryCode as string,
        )
      ) : (
        <span className="flex items-center gap-1 italic font-medium text-slate-400">
          <PhoneOff size={16} /> {t("common.no_phone_data")}
        </span>
      ),
      colorClass:
        "bg-emerald-100 text-emerald-600 dark:bg-emerald-500/15 dark:text-emerald-400",
    },
    {
      id: "birthday",
      icon: <Cake size={isMobile ? 15 : 20} />,
      label: t("common.birthday"),
      value: profile?.birthdayDate ? (
        DDMMMMYYYY(profile?.birthdayDate)
      ) : (
        <span className="flex items-center gap-1 italic font-medium text-slate-400">
          <CalendarMinus size={16} /> {t("common.no_birthday_data")}
        </span>
      ),
      colorClass:
        "bg-orange-100 text-orange-600 dark:bg-orange-500/15 dark:text-orange-400",
    },
    {
      id: "location",
      icon: <MapPin size={isMobile ? 15 : 20} />,
      label: t("common.location"),
      value: profile?.location || (
        <span className="flex items-center gap-1 italic font-medium text-slate-400">
          <MapMinusIcon size={16} /> {t("common.no_location_info")}
        </span>
      ),
      colorClass:
        "bg-emerald-100 text-emerald-600 dark:bg-emerald-500/15 dark:text-emerald-400",
    },
  ];

  return (
    <Fragment>
      <section
        className="p-8 mb-8 px-4 md:px-8 transition-all duration-300 bg-(--auth-main-bg) rounded-[20px] border border-(--auth-glass-border) shadow-(--card-shadow)"
        aria-labelledby="personal-info-heading"
      >
        {/* Header */}
        <div className="flex flex-row items-center justify-between gap-3 mb-6 md:mb-8">
          {/* Sol tərəf: İkon və Başlıq */}
          <div className="flex items-center gap-2 md:gap-3">
            <Contact className="w-6 h-6 md:w-7 md:h-7 text-(--primary-color)" />
            <h2
              id="personal-info-heading"
              className="m-0 text-lg font-bold md:text-2xl text-(--text-color)"
            >
              {t("common.personal_information")}
            </h2>
          </div>
        </div>

        {/* Grid */}

        <div className="grid grid-cols-1 gap-4 sm:grid-cols-2">
          {infoItems.map((item) => (
            <div
              key={item.id}
              className="flex items-center gap-4 p-4 lg:px-5 bg-(--info-card-bg) border border-(--info-card-border) rounded-xl transition-all duration-200 group"
            >
              {/* Icon Box */}
              <div
                className={`flex items-center justify-center rounded-xl shrink-0 transition-transform w-10 h-10 md:w-12 md:h-12 ${item.colorClass}`}
              >
                {item.icon}
              </div>

              {/* Content */}
              <div className="flex flex-col flex-1 gap-1 overflow-hidden">
                <span className="text-[11px] font-bold tracking-widest text-(--muted-color) uppercase">
                  {item.label}
                </span>
                <div className="text-[15px] font-semibold text-(--text-color) truncate flex items-center gap-1.5">
                  {item.value}
                </div>
              </div>
            </div>
          ))}
        </div>
      </section>
    </Fragment>
  );
};

export default PersonalInformation;

```

## File: connectfy-client/src/modules/users/UserProfile/ui/components/MainCard/MainCard.tsx
```typescript
import {
  Users,
  MessageCircle,
  UserPlus,
  Clock,
  UserCheck,
  Check,
  X,
} from "lucide-react";
import { useTranslation } from "react-i18next";
import { FC, Fragment } from "react";
import NoProfilePhotoIcon from "@/assets/icons/NoProfilePhotoIcon";
import {
  IFindOneActions,
  IFindOneProfile,
  IFindOneRelationship,
  IFindOneUser,
} from "../../../types/types";
import { useGeneralSettings } from "@/modules/settings/GeneralSettings/hooks/useGeneralSettings";
import { showDate } from "@/common/utils/formatValues";
import { DATE_FORMAT, FriendshipStatus } from "@/common/enums/enums";
import Button from "@/components/ui/CustomButton/Button/Button";
import useBoolean from "@/hooks/useBoolean";
import AvatarModal from "../Modal/AvatarModal/AvatarModal";
import MutualFriendsModal from "../Modal/MutualFriendsModal/MutualFriendsModal";
import { useFriendship } from "@/modules/users/MyFriends/hooks/useFriendship";
import { useFindMutualFriendsQuery } from "@/modules/users/MyFriends/api/api";

interface IProps {
  user: IFindOneUser;
  profile: IFindOneProfile;
  actions: IFindOneActions;
  relationship: IFindOneRelationship;
}

const MainCard: FC<IProps> = ({ user, profile, actions, relationship }) => {
  const { t } = useTranslation();
  const { generalSettings } = useGeneralSettings();

  const openAvatarModal = useBoolean();
  const openMutualFriendsModal = useBoolean();

  const status = relationship?.friendship?.status;
  const senderId = relationship?.friendship?.userId;

  const { data: mutualFriendsData } = useFindMutualFriendsQuery({
    targetUserId: user._id,
    skip: 1,
    limit: 1,
  });

  const {
    handleSendFriendRequest,
    handleAcceptFriendRequest,
    handleDeclineFriendRequest,
    handleCancelFriendRequest,
    handleRemoveFriendship,
    isActionLoading,
  } = useFriendship();

  // Dostluq istəyini qarşı tərəfin (profil sahibinin) göndərdiyini yoxlayırıq
  const isPendingReceiver =
    status === FriendshipStatus.Pending && senderId === user._id;

  // Dostluq statusuna görə düyməni render edən köməkçi funksiya
  const renderFriendActionButton = () => {
    // 1. Əgər dostluq yoxdursa
    if (!relationship?.friendship) {
      return (
        <Button
          type="button"
          disabled={!actions?.canSendFriendRequest || isActionLoading}
          className="px-6 py-2.5 text-sm font-semibold text-(--primary-color) transition-colors rounded-xl bg-(--active-bg-2)"
          icon={<UserPlus size={18} />}
          title={t("common.add_friend")}
          hideTitleInMobile={false}
          isLoading={isActionLoading}
          onClick={() => handleSendFriendRequest({ friendId: user?._id || "" })}
        />
      );
    }

    // 2. Əgər PENDİNG (gözləmədə) vəziyyətindədirsə
    if (status === FriendshipStatus.Pending) {
      return (
        <Button
          type="button"
          className="px-6 py-2.5 text-sm font-semibold text-(--text-color) transition-colors rounded-xl bg-(--active-bg-2)"
          icon={<Clock size={18} />}
          title={t("common.pending")}
          disabled={senderId === user._id || isActionLoading}
          hideTitleInMobile={false}
          isLoading={isActionLoading}
          onClick={() =>
            handleCancelFriendRequest({
              friendshipId: relationship?.friendship?._id || "",
              userId: user._id || "",
            })
          }
        />
      );
    }

    // 3. Əgər ACCEPTED (dostluq qəbul edilibsə) vəziyyətindədirsə
    if (status === FriendshipStatus.Accepted) {
      return (
        <Button
          type="button"
          className="px-6 py-2.5 text-sm font-semibold text-(--primary-color) transition-colors rounded-xl bg-(--active-bg-2)"
          icon={<UserCheck size={18} />}
          title={t("common.friends")}
          hideTitleInMobile={false}
          disabled={isActionLoading}
          isLoading={isActionLoading}
          onClick={() =>
            handleRemoveFriendship({
              friendshipId: relationship?.friendship?._id || "",
              userId: user._id || "",
            })
          }
        />
      );
    }

    return null;
  };

  return (
    <Fragment>
      {/* Dostluq istəyi gəldiyi zaman görünən Bildiriş Kartı */}
      {isPendingReceiver && (
        <div className="flex flex-col items-center justify-between gap-4 p-5 mb-6 transition-all sm:flex-row rounded-[20px] bg-(--active-bg-2) border border-(--shadow-color) shadow-(--card-shadow)">
          <div className="text-center sm:text-left">
            <p className="m-0 text-sm font-medium text-(--text-color)">
              <span className="font-bold text-(--text-primary)">
                {profile?.fullName || `${profile?.firstName} ${profile?.lastName}`}
              </span>{" "}
              {t("common.sent_friend_request")}
            </p>
          </div>
          <div className="flex items-center w-full gap-3 sm:w-auto">
            <Button
              type="button"
              className="flex-1 px-5 py-2 text-sm font-semibold text-white transition-colors rounded-xl sm:flex-none bg-(--primary-color) hover:bg-(--hover-bg)"
              icon={<Check size={18} />}
              title={t("common.accept")}
              hideTitleInMobile={false}
              disabled={isActionLoading}
              isLoading={isActionLoading}
              onClick={() =>
                handleAcceptFriendRequest({
                  friendshipId: relationship?.friendship?._id || "",
                  userId: user._id || "",
                })
              }
            />
            <Button
              type="button"
              className="flex-1 px-5 py-2 text-sm font-semibold transition-colors rounded-xl sm:flex-none text-(--text-color) bg-(--input-bg) hover:bg-(--skeleton-card-bg)"
              icon={<X size={18} />}
              title={t("common.decline")}
              hideTitleInMobile={false}
              disabled={isActionLoading}
              isLoading={isActionLoading}
              onClick={() =>
                handleDeclineFriendRequest({
                  friendshipId: relationship?.friendship?._id || "",
                  userId: user._id || "",
                })
              }
            />
          </div>
        </div>
      )}

      <section
        className="flex flex-col items-center gap-6 p-8 mb-8 transition-all bg-(--auth-main-bg) rounded-[20px] border border-(--auth-glass-border) shadow-(--card-shadow) md:p-10"
        aria-labelledby="profile-main-heading"
      >
        <div className="relative shrink-0">
          <div className="size-[140px] xs:size-[100px] sm:size-[120px] md:size-[140px] rounded-full overflow-hidden shadow-[0_8px_24px_var(--shadow-color)] border-4 border-(--primary-color)">
            {profile?.avatar?.url ? (
              <img
                src={profile.avatar.url}
                alt={`Profile picture of ${profile?.fullName || `${profile?.firstName} ${profile?.lastName}`}`}
                loading="eager"
                fetchPriority="high"
                decoding="async"
                className="object-cover w-full h-full bg-(--skeleton-card-bg) cursor-pointer"
                onClick={() => openAvatarModal.onOpen()}
              />
            ) : (
              <div className="flex items-center justify-center w-full h-full bg-(--active-bg-2)">
                <NoProfilePhotoIcon />
              </div>
            )}
          </div>
        </div>

        <div className="text-center">
          <h1
            id="profile-main-heading"
            className="m-0 mb-2 text-2xl font-bold leading-tight md:text-[28px] text-(--text-color)"
          >
            {profile?.fullName || `${profile?.firstName} ${profile?.lastName}`}
          </h1>
          <p className="m-0 mb-2 text-sm font-medium md:text-base text-(--muted-color)">
            @{user?.username}
          </p>
          {profile?.lastSeen && (
            <p className="m-0 text-sm italic text-(--muted-color)">
              {t("common.lastSeen")}:{" "}
              {showDate(
                profile.lastSeen,
                generalSettings?.timeZone.dateFormat || DATE_FORMAT.DDMMYYYY,
                "/",
              )}
            </p>
          )}
        </div>

        <div className="flex flex-col items-center w-full max-w-[400px] gap-4 p-4 rounded-2xl bg-(--active-bg-2) sm:flex-row sm:gap-6 sm:p-6 md:px-8">
          <div className="flex flex-row items-center justify-center flex-1 gap-3 sm:flex-col sm:gap-1.5 text-(--text-color)">
            <div className="flex items-center gap-1.5">
              <Users
                size={18}
                className="text-(--primary-color)"
                aria-hidden="true"
              />
              <span className="text-xl font-bold md:text-2xl text-(--primary-color)">
                {relationship?.count || 0}
              </span>
            </div>
            <span className="text-[13px] font-medium uppercase tracking-wider text-(--muted-color)">
              {t("common.friends")}
            </span>
          </div>
          <div className="w-[1px] h-10 bg-(--shadow-color) hidden sm:block"></div>
          <div 
            className="flex flex-row items-center justify-center flex-1 gap-3 sm:flex-col sm:gap-1.5 text-(--text-color) cursor-pointer group"
            onClick={() => openMutualFriendsModal.onOpen()}
          >
            <div className="flex items-center gap-1.5">
              <Users
                size={18}
                className="text-(--muted-color) group-hover:text-(--primary-color) transition-colors"
                aria-hidden="true"
              />
              <span className="text-xl font-bold md:text-2xl text-(--text-color) group-hover:text-(--primary-color) transition-colors">
                {mutualFriendsData?.count || 0}
              </span>
            </div>
            <span className="text-[13px] font-medium uppercase tracking-wider text-(--muted-color) group-hover:text-(--primary-color) transition-colors">
              {t("common.mutual_friends", "Mutual Friends")}
            </span>
          </div>
        </div>

        {/* Action Buttons Section */}
        <div className="flex flex-col w-full gap-3 mt-2 sm:flex-row sm:justify-center sm:max-w-[400px]">
          {/* Message Button */}
          <Button
            type="button"
            disabled={!actions?.canSendMessage}
            className="px-6 py-2.5 w-full md:w-1/2 text-sm font-semibold text-(--text-color) transition-colors rounded-xl bg-(--active-bg-2)"
            icon={<MessageCircle size={18} />}
            title={t("common.message")}
            hideTitleInMobile={false}
          />

          {/* Add Friend / Status Button */}
          <div className="flex flex-col flex-1">
            {renderFriendActionButton()}
          </div>
        </div>
      </section>

      <AvatarModal
        open={openAvatarModal.open}
        onClose={openAvatarModal.onClose}
        avatarUrl={profile?.avatar?.url ?? ""}
        username={user.username}
        userId={user._id}
      />
      <MutualFriendsModal
        open={openMutualFriendsModal.open}
        onClose={openMutualFriendsModal.onClose}
        targetUserId={user._id}
      />
    </Fragment>
  );
};

export default MainCard;

```

## File: connectfy-client/src/modules/users/UserProfile/ui/components/ProfileHeader/ProfileHeader.tsx
```typescript
import { FC, Fragment, useMemo } from "react";
import { useTranslation } from "react-i18next";
import {
  ArrowLeft,
  User,
  UserLock,
  BellRing,
  BellOff,
  Share,
  Star,
  UserMinus,
  UserPlus,
  ChevronDown,
} from "lucide-react";
import { FriendshipStatus } from "@/common/enums/enums";
import { useAppNavigation } from "@/hooks/useAppNavigation";
import { useIsMobile } from "@/hooks/useIsMobile";
import Button from "@/components/ui/CustomButton/Button/Button";
import Dropdown, {
  DropdownOption,
} from "@/components/ui/Select/Dropdown/Dropdown";
import { IFindOneRelationship, IFindOneUser, IFindOneActions } from "../../../types/types";
import { useFriendship } from "@/modules/users/MyFriends/hooks/useFriendship";
import { useBlocklist } from "@/modules/users/Blocklist/hooks/useBlocklist";
import useBoolean from "@/hooks/useBoolean";
import ShareModal from "../Modal/ShareModal/ShareModal";

interface IProps {
  relationship?: IFindOneRelationship;
  user?: IFindOneUser;
  actions?: IFindOneActions;
}

const ProfileHeader: FC<IProps> = ({ relationship, user, actions }) => {
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();
  const isMobile = useIsMobile();

  const {
    handleSendFriendRequest,
    handleRemoveFriendship,
    handleUpdateCloseFriend,
    handleUpdateNotification,

    isActionLoading,
    isUpdateCloseFriendLoading,
    isUpdateNotificationLoading,
  } = useFriendship();

  const { handleBlockUser, handleUnblockUser, isBlockUserLoading, isUnblockUserLoading } = useBlocklist();

  const shareModal = useBoolean();

  // Dostluq statuslarını müəyyən edirik
  const isFriend =
    relationship?.friendship?.status === FriendshipStatus.Accepted;
  const isPending =
    relationship?.friendship?.status === FriendshipStatus.Pending;

  const profileUrl = `${window.location.origin}/users/profile/${user?._id}`;

  const dropdownOptions: DropdownOption[] = useMemo(() => {
    const options: DropdownOption[] = [];

    if (actions?.isBlocked) {
      if (actions?.hasBlocked) {
        options.push({
          label: t("common.unblock", { defaultValue: "Unblock User" }),
          value: "block_user",
          icon: <UserLock size={20} />,
          className: "text-(--error-color)!",
          onClick: () => handleUnblockUser(user?._id || ""),
        });
      }
      return options;
    }

    if (isMobile) {
      if (isFriend) {
        options.push(
          {
            label: relationship?.friendship?.isMuted
              ? t("common.unmute")
              : t("common.notifications"),
            value: "notifications",
            icon: relationship?.friendship?.isMuted ? (
              <BellOff size={20} />
            ) : (
              <BellRing size={20} />
            ),
            onClick: () =>
              handleUpdateNotification({
                userId: user?._id || "",
                friendshipId: relationship?.friendship?._id || "",
                notification: relationship?.friendship?.isMuted || false,
              }),
            className: "text-(--primary-color)!",
          },
          {
            label: relationship?.friendship?.isFavorite
              ? t("common.remove_close_friend")
              : t("common.close_friend"),
            value: "close_friend",
            icon: (
              <Star
                size={20}
                className={
                  relationship?.friendship?.isFavorite
                    ? "fill-(--text-color)"
                    : ""
                }
              />
            ),
            onClick: () =>
              handleUpdateCloseFriend({
                userId: user?._id || "",
                friendshipId: relationship?.friendship?._id || "",
                isCloseFriend: !relationship?.friendship?.isFavorite,
              }),
            className: "text-(--primary-color)!",
          },
        );
      } else {
        options.push({
          label: isPending ? t("common.pending") : t("common.add_friend"),
          value: "add_friend",
          icon: <UserPlus size={20} />,
          className: isPending
            ? "opacity-50 cursor-not-allowed pointer-events-none"
            : "text-(--primary-color)!",
          onClick: () => {
            if (!isPending)
              handleSendFriendRequest({ friendId: user?._id || "" });
          },
        });
      }

      options.push({
        label: t("common.share"),
        value: "share",
        icon: <Share size={20} />,
        className: "text-(--primary-color)!",
        onClick: shareModal.onOpen,
      });

      options.push({
        label: t("common.block_user"),
        value: "block_user",
        icon: <UserLock size={20} />,
        className: "text-(--error-color)!",
        onClick: () => handleBlockUser(user?._id || ""),
      });

      if (isFriend) {
        options.push({
          label: t("common.unfriend"),
          value: "unfriend",
          icon: <UserMinus size={20} />,
          className: "text-(--error-color)!",
          onClick: () =>
            handleRemoveFriendship({
              friendshipId: relationship?.friendship?._id || "",
              userId: user?._id || "",
            }),
        });
      }
    } else {
      if (isFriend) {
        options.push(
          {
            label: t("common.share"),
            value: "share",
            icon: <Share size={20} />,
            className: "text-(--primary-color)!",
            onClick: shareModal.onOpen,
          },
          {
            label: t("common.unfriend"),
            value: "unfriend",
            icon: <UserMinus size={20} />,
            className: "text-(--error-color)!",
            onClick: () =>
              handleRemoveFriendship({
                friendshipId: relationship?.friendship?._id || "",
                userId: user?._id || "",
              }),
          },
          {
            label: t("common.block_user"),
            value: "block_user",
            icon: <UserLock size={20} />,
            className: "text-(--error-color)!",
            onClick: () => handleBlockUser(user?._id || ""),
          },
        );
      }
    }

    return options;
  }, [
    isMobile,
    isFriend,
    isPending,
    relationship,
    user,
    t,
    shareModal.onOpen,
    handleUpdateNotification,
    handleUpdateCloseFriend,
    handleRemoveFriendship,
    handleSendFriendRequest,
    handleBlockUser,
    handleUnblockUser,
    actions?.hasBlocked,
    navigate,
  ]);

  const baseActionBtnClass = `
    flex items-center justify-center gap-2
    px-4 h-10 min-h-10 rounded-xl
    border-none cursor-pointer font-medium text-sm whitespace-nowrap
    shadow-(--card-shadow)
    transition-all duration-300 ease-in-out
    hover:-translate-y-0.5 hover:shadow-(--active-shadow)
    active:translate-y-0
    focus-visible:outline-2 focus-visible:-outline-offset-0.5
    max-[768px]:px-3 max-[768px]:h-9 max-[768px]:min-h-9
    max-[480px]:px-2.5 max-[480px]:h-8 max-[480px]:min-h-8 max-[480px]:gap-1.5
    lg:w-[150px]
    disabled:opacity-50 disabled:cursor-not-allowed disabled:hover:translate-y-0 disabled:hover:shadow-(--card-shadow)
  `;

  const primaryBtnClass = `
    ${baseActionBtnClass}
    bg-[rgba(var(--primary-color-rgb),0.1)] text-(--text-color)
    hover:bg-(--active-bg)
    focus-visible:outline-(--primary-color)
    disabled:hover:bg-[rgba(var(--primary-color-rgb),0.1)] disabled:hover:text-(--primary-color)
  `;

  const errorBtnClass = `
    ${baseActionBtnClass}
    bg-[rgba(var(--error-color-rgb),0.1)] text-(--error-color)
    hover:bg-(--error-color) hover:text-white
    focus-visible:outline-(--error-color)
  `;

  return (
    <header
      className="
        sticky top-0 left-0 right-0 z-100
        flex items-center justify-between
        gap-5 px-4 min-h-[72px]
        bg-(--auth-main-bg)
        backdrop-blur-[10px]
        max-sm:px-3 max-sm:gap-3 max-sm:min-h-[64px]
      "
    >
      {/* Left */}
      <div className="flex items-center gap-4 flex-1 min-w-0 max-sm:gap-3 max-sm:flex-none">
        <Button
          aria-label={t("common.back")}
          onClick={() => navigate(-1)}
          icon={<ArrowLeft size={18} />}
          className="
            flex items-center justify-center shrink-0 p-0
            w-[43px] h-[43px] rounded-[10px]
            bg-transparent text-(--text-color) cursor-pointer
            transition-all duration-300 ease-in-out
            hover:bg-(--active-bg) hover:border-(--primary-color) hover:-translate-x-0.5
            active:translate-x-0
            focus-visible:outline-2 focus-visible:outline-(--primary-color) focus-visible:-outline-offset-0.5
            max-[768px]:w-9 max-[768px]:h-9 max-[768px]:min-w-9 max-[768px]:min-h-9
            max-[480px]:w-8 max-[480px]:h-8 max-[480px]:min-w-8 max-[480px]:min-h-8 max-[480px]:rounded-lg
          "
        />
        <h1
          className="
            flex items-center gap-2.5 m-0 p-0
            text-xl font-bold text-(--text-color)
            whitespace-nowrap overflow-hidden text-ellipsis
            [&>svg]:text-(--primary-color) [&>svg]:stroke-[2.5] [&>svg]:shrink-0 [&>svg]:w-[18px]
            max-[768px]:text-lg max-[768px]:gap-2
            max-[640px]:text-base max-[640px]:shrink
            max-[480px]:text-base max-[480px]:gap-1.5 max-[480px]:[&>svg]:w-4 max-[480px]:[&>svg]:h-4
          "
        >
          <User size={20} />
          <span>{user?.username ?? t("common.profile")}</span>
        </h1>
      </div>

      {/* Right */}
      <div className="flex items-center gap-3 max-sm:flex-1 max-sm:justify-end max-sm:min-w-0">
        {!isMobile && actions?.isBlocked && actions?.hasBlocked && (
          <Button
            onClick={() => handleUnblockUser(user?._id || "")}
            icon={<UserLock size={20} />}
            title={t("common.unblock", { defaultValue: "Unblock User" })}
            className={errorBtnClass}
            isLoading={isUnblockUserLoading}
            disabled={isUnblockUserLoading}
          />
        )}

        {!isMobile && !actions?.isBlocked && !isFriend && (
          <Fragment>
            <Button
              icon={<UserPlus size={20} />}
              title={isPending ? t("common.pending") : t("common.add_friend")}
              className={primaryBtnClass}
              disabled={isPending || isActionLoading}
              onClick={() =>
                handleSendFriendRequest({ friendId: user?._id || "" })
              }
            />
            <Button
              onClick={() => handleBlockUser(user?._id || "")}
              icon={<UserLock size={20} />}
              title={t("common.block_user")}
              className={errorBtnClass}
              isLoading={isBlockUserLoading}
              disabled={isBlockUserLoading}
            />
            <Button
              icon={<Share size={20} />}
              title={t("common.share")}
              className={primaryBtnClass}
              onClick={shareModal.onOpen}
            />
          </Fragment>
        )}

        {!isMobile && !actions?.isBlocked && isFriend && (
          <Fragment>
            <Button
              icon={
                relationship?.friendship?.status ===
                  FriendshipStatus.Accepted &&
                relationship?.friendship?.isMuted ? (
                  <BellOff size={20} />
                ) : (
                  <BellRing size={20} />
                )
              }
              title={t("common.notifications")}
              className={primaryBtnClass}
              disabled={isUpdateNotificationLoading}
              isLoading={isUpdateNotificationLoading}
              onClick={() =>
                handleUpdateNotification({
                  userId: user?._id || "",
                  friendshipId: relationship?.friendship?._id || "",
                  notification: relationship?.friendship?.isMuted || false,
                })
              }
            />
            <Button
              icon={
                <Star
                  size={20}
                  className={`
                    transition-colors duration-300
                    ${relationship?.friendship?.isFavorite ? "fill-(--text-color)" : ""}
                  `}
                />
              }
              title={t("common.close_friend")}
              className={`group ${primaryBtnClass}`}
              disabled={isUpdateCloseFriendLoading}
              isLoading={isUpdateCloseFriendLoading}
              onClick={() =>
                handleUpdateCloseFriend({
                  userId: user?._id || "",
                  friendshipId: relationship?.friendship?._id || "",
                  isCloseFriend: !relationship?.friendship?.isFavorite,
                })
              }
            />
          </Fragment>
        )}

        {/* DROPDOWN (Mobildə həmişə çıxacaq, Desktopda isə yalnız dost olduqda çıxacaq) */}
        {(isMobile || (!isMobile && isFriend)) && dropdownOptions.length > 0 && (
          <Dropdown
            options={dropdownOptions}
            icon={<ChevronDown size={20} className="text-inherit" />}
            text={!isMobile ? t("common.more") : undefined}
            animation={{
              whileHover: {},
              whileTap: { scale: 0.95 },
              animate: { rotate: 180 },
            }}
            buttonClassName="!gap-2 !px-4 !w-auto !h-10 !min-h-10 !rounded-xl
            border-none cursor-pointer font-medium text-sm whitespace-nowrap
            shadow-(--card-shadow)
            bg-[rgba(var(--primary-color-rgb),0.1)]!
            transition-all duration-300 ease-in-out
            hover:bg-(--active-bg)! hover:text-(--text-color)! hover:-translate-y-0.5 hover:shadow-(--active-shadow)
            active:translate-y-0
            focus-visible:outline-2 focus-visible:outline-(--primary-color) focus-visible:-outline-offset-0.5
            max-[768px]:!px-3 max-[768px]:!h-9 max-[768px]:!min-h-9
            max-[480px]:!px-2.5 max-[480px]:!h-8 max-[480px]:!min-h-8 max-[480px]:gap-1.5 lg:!w-[150px] origin-top"
            openAnimation={{
              initial: { opacity: 0, scale: 0.9, y: -10 },
              animate: { opacity: 1, scale: 1, y: 0 },
              exit: { opacity: 0, scale: 0.9, y: -10 },
              transition: { duration: 0.15, ease: "easeOut" },
            }}
          />
        )}
      </div>

      <ShareModal
        open={shareModal.open}
        onClose={shareModal.onClose}
        profileUrl={profileUrl}
        username={user?.username || ""}
      />
    </header>
  );
};

export default ProfileHeader;

```

## File: connectfy-client/src/modules/users/UserProfile/ui/components/SocialLinks/SocialLinks.tsx
```typescript
import { FC, Fragment, useCallback, useMemo, useState } from "react";
import { useTranslation } from "react-i18next";
import { Link2 } from "lucide-react";
import { Reorder } from "framer-motion";

import { useAuthStore } from "@/store/zustand/useAuthStore";
import { ISocialLink } from "@/modules/profile/types/types";
import { snack } from "@/common/utils/snackManager";

import { useGetSocialLinksQuery } from "@/modules/profile/api/api";

import SocialLinkSkeleton from "@/components/Skeleton/profile/SocialLinkSkeleton";

import SocialLinkItem from "./components/SocialLinkItem";
import SocialLinkHeader from "./components/SocialLinkHeader";
import { useContextMenu } from "@/hooks/useContextMenu";
import SocialContextMenu from "./components/SocialContextMenu";

interface IProps {
  userId: string | undefined;
}

const SocialLinks: FC<IProps> = ({ userId }) => {
  const { t } = useTranslation();
  const { access_token } = useAuthStore();

  // States
  const [activeFilter, setActiveFilter] = useState<
    "rank" | "newest" | "oldest"
  >("rank");
  const [_, setReorderLinks] = useState<ISocialLink[]>([]);

  const { handleContextMenu } = useContextMenu();

  // API Hooks
  const { data: socialLinks, isLoading } = useGetSocialLinksQuery(
    { userId: userId || "" },
    { skip: !access_token || !userId },
  );

  const sortedData = useMemo(() => {
    if (!socialLinks?.data) return [];

    const links = [...socialLinks.data];

    switch (activeFilter) {
      case "newest":
        return links.sort(
          (a, b) =>
            new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime(),
        );
      case "oldest":
        return links.sort(
          (a, b) =>
            new Date(a.createdAt).getTime() - new Date(b.createdAt).getTime(),
        );
      case "rank":
      default:
        return links.sort((a, b) => (a.rank || 0) - (b.rank || 0));
    }
  }, [socialLinks?.data, activeFilter]);

  const orderedLinks = sortedData;

  const handleSocialAction = useCallback(
    (action: string, link: ISocialLink) => {
      switch (action) {
        case "openLink":
          window.open(link.url, "_blank");
          break;
        case "copy":
          navigator.clipboard.writeText(link.url);
          snack.success(t("common.link_copied"));
          break;
      }
    },
    [t],
  );

  const hasLinks = orderedLinks && orderedLinks.length > 0;

  return (
    <Fragment>
      {isLoading ? (
        <SocialLinkSkeleton />
      ) : (
        <section
          className="p-8 mb-8 transition-all duration-300 bg-(--auth-main-bg) rounded-[20px] border border-(--auth-glass-border) shadow-(--card-shadow)"
          aria-labelledby="social-links-heading"
        >
          {/* Ekstrakt edilmiş Header Komponenti */}
          <SocialLinkHeader
            activeFilter={activeFilter}
            onFilterChange={setActiveFilter}
          />

          {/* Siyahı hissəsi */}
          {hasLinks ? (
            <Reorder.Group
              axis="y"
              values={orderedLinks}
              onReorder={setReorderLinks}
              className="flex flex-col gap-4"
            >
              {orderedLinks.map((link, index) => {
                return (
                  <Reorder.Item
                    key={link._id}
                    value={link}
                    className={"flex items-center w-full"}
                    onContextMenu={(e) => {
                      handleContextMenu(
                        e,
                        <SocialContextMenu
                          onCopy={() => handleSocialAction("copy", link)}
                          onOpenLink={() =>
                            handleSocialAction("openLink", link)
                          }
                        />,
                      );
                    }}
                  >
                    <div className="flex-1 min-w-0 transition-all duration-300">
                      <div>
                        <SocialLinkItem
                          link={link}
                          onAction={handleSocialAction}
                          index={index}
                        />
                      </div>
                    </div>
                  </Reorder.Item>
                );
              })}
            </Reorder.Group>
          ) : (
            <div className="flex flex-col items-center justify-center p-10 border border-dashed rounded-xl border-white/10 bg-black/5">
              <Link2
                className="mb-3 opacity-20 text-(--text-color)"
                size={48}
              />
              <span className="text-(--muted-color) italic font-medium">
                {t("common.no_social_links")}
              </span>
            </div>
          )}
        </section>
      )}
    </Fragment>
  );
};

export default SocialLinks;

```

## File: connectfy-client/src/modules/users/UserProfile/ui/components/SocialLinks/components/SocialLinkItem.tsx
```typescript
import { Link } from "react-router-dom";
import { ChevronDown, ChevronUp, ExternalLink, Copy } from "lucide-react";
import { memo, useState } from "react";
import { ISocialLink } from "@/modules/profile/types/types";
import Button from "@/components/ui/CustomButton/Button/Button";
import { useTranslation } from "react-i18next";
import { PLATFORM_ICONS } from "./SocialPlatformIcons";
import { SOCIAL_LINK_PLATFORM } from "@/common/enums/enums";

// Social Link Item Component
const SocialLinkItem = memo(
  ({
    link,
    onAction,
    index,
  }: {
    link: ISocialLink;
    onAction: (action: string, link: ISocialLink) => void;
    index: number;
  }) => {
    const { t } = useTranslation();
    const [isExpanded, setIsExpanded] = useState(false);

    const handleCopy = (e: React.MouseEvent) => {
      e.stopPropagation();
      navigator.clipboard.writeText(link.url);
      onAction("copy", link);
    };

    return (
      <div
        className={
          "relative overflow-hidden transition-all duration-300 border rounded-xl group bg-(--info-card-bg) border-(--info-card-border) hover:border-(--primary-color)"
        }
        data-testid={`social-link-item-${index}`}
      >
        {/* Accent Bar (Hover zamanı solda görünən rəngli xətt) */}
        <div
          className={
            "absolute top-0 left-0 w-1 h-full transition-opacity bg-(--primary-color) opacity-0 group-hover:opacity-100"
          }
        />

        {/* Header Button */}
        <Button
          onClick={() => {
            setIsExpanded(!isExpanded);
          }}
          className="flex items-center justify-between w-full px-5 py-4 text-left transition-colors hover:bg-(--active-bg-2)"
          aria-expanded={isExpanded}
        >
          <div className="flex items-center gap-3">
            <div
              className={
                "transition-colors shrink-0 text-(--muted-color) group-hover:text-(--primary-color)"
              }
            >
              {PLATFORM_ICONS[link.platform as SOCIAL_LINK_PLATFORM]}
            </div>
            <div className="flex flex-col gap-1">
              <span className="text-[15px] font-bold text-(--text-color) tracking-wide">
                {link.platform}
              </span>
              <span className="text-sm font-medium text-(--muted-color)">
                {link.name}
              </span>
            </div>
          </div>
          {/* DƏYİŞİKLİK: Selection mode-da chevron-u gizlədə və ya checkbox göstərə bilərsən. 
              Amma sadəlik üçün chevron-u seçim zamanı gizlətmək daha yaxşıdır. */}

          <div className="text-(--muted-color) group-hover:text-(--text-color) transition-colors">
            {isExpanded ? <ChevronUp size={20} /> : <ChevronDown size={20} />}
          </div>
        </Button>

        {/* Details (Expanded Content) */}
        <div
          className={`overflow-hidden transition-all duration-300 ease-in-out ${
            isExpanded ? "max-h-80 opacity-100" : "max-h-0 opacity-0"
          }`}
        >
          <div className="p-5 pt-0 border-t border-(--info-card-border)/50">
            {/* URL Display */}
            <Link
              to={link.url}
              target="_blank"
              rel="noopener noreferrer"
              className="block p-3 mt-4 text-sm font-medium transition-all border border-dashed rounded-lg border-white/10 text-(--primary-color) bg-white/70 dark:bg-black/30 truncate"
            >
              {link.url}
            </Link>

            {/* Action Buttons */}
            <div className="flex flex-wrap gap-2 mt-4">
              <ActionButton
                onClick={() => onAction("openLink", link)}
                icon={<ExternalLink size={16} />}
                label={t("common.open")}
              />
              <ActionButton
                onClick={handleCopy}
                icon={<Copy size={16} />}
                label={t("common.copy")}
              />
            </div>
          </div>
        </div>
      </div>
    );
  },
);

// Köməkçi Düymə Komponenti (Daxili düymələr üçün)
const ActionButton = ({ onClick, icon, label, isDelete }: any) => (
  <Button
    onClick={onClick}
    className={`flex-1 min-w-[100px] flex items-center justify-center gap-2 px-3 py-2.5 rounded-lg text-sm font-semibold transition-all border
      ${
        isDelete
          ? "border-red-500/30 text-red-500 hover:bg-red-500 hover:text-white"
          : "border-white/5 bg-white/5 text-(--text-color) hover:bg-(--primary-color) hover:text-white hover:scale-[1.02]"
      }`}
  >
    {icon}
    <span className="hidden sm:inline">{label}</span>
  </Button>
);

export default SocialLinkItem;

```

## File: connectfy-client/src/modules/users/UserProfile/ui/components/SocialLinks/components/SocialLinkHeader.tsx
```typescript
import { FC, useMemo, Fragment } from "react";
import { useTranslation } from "react-i18next";
import { motion, AnimatePresence } from "framer-motion";
import { Link2, ListFilter, MoreVertical } from "lucide-react";

import Dropdown, {
  DropdownOption,
} from "@/components/ui/Select/Dropdown/Dropdown";
import { useIsMobile } from "@/hooks/useIsMobile";

interface ISocialLinkHeaderProps {
  activeFilter: "rank" | "newest" | "oldest";
  onFilterChange: (filter: "rank" | "newest" | "oldest") => void;
}

export const SocialLinkHeader: FC<ISocialLinkHeaderProps> = ({
  activeFilter,
  onFilterChange,
}) => {
  const { t } = useTranslation();
  const isMobile = useIsMobile();

  const filterOptions: DropdownOption[] = useMemo(
    () => [
      {
        label: t("common.by_rank"),
        value: "rank",
        isActive: activeFilter === "rank",
        onClick: () => onFilterChange("rank"),
      },
      {
        label: t("common.by_newest"),
        value: "newest",
        isActive: activeFilter === "newest",
        onClick: () => onFilterChange("newest"),
      },
      {
        label: t("common.by_oldest"),
        value: "oldest",
        isActive: activeFilter === "oldest",
        onClick: () => onFilterChange("oldest"),
      },
    ],
    [t, onFilterChange, activeFilter],
  );

  const mobileMoreOptions: DropdownOption[] = useMemo(() => {
    const options: DropdownOption[] = [];

    options.push({
      label: t("common.filter"),
      value: "filter",
      icon: <ListFilter size={16} />,
      subMenu: filterOptions,
    });

    return options;
  }, [filterOptions, t]);

  return (
    <div className="flex flex-row items-center justify-between gap-3 mb-6 md:mb-8 min-h-[40px]">
      <div className="flex items-center gap-2 md:gap-3">
        <Link2 className="w-6 h-6 md:w-7 md:h-7 text-(--primary-color)" />
        <h2 className="m-0 text-lg font-bold md:text-2xl text-(--text-color)">
          {t("common.social_links")}
        </h2>
      </div>

      <div className="flex items-center gap-2 md:gap-3 relative">
        <AnimatePresence mode="wait">
          <motion.div
            key="default"
            initial={{ opacity: 0, x: -20 }}
            animate={{ opacity: 1, x: 0 }}
            exit={{ opacity: 0, x: -20 }}
            className="flex items-center gap-2 md:gap-3"
          >
            {isMobile ? (
              <Dropdown
                options={mobileMoreOptions}
                icon={<MoreVertical size={20} />}
                tooltip={t("common.more")}
              />
            ) : (
              <Fragment>
                <Dropdown
                  options={filterOptions}
                  selected={activeFilter}
                  icon={<ListFilter size={18} />}
                  tooltip={t("common.filter")}
                />
              </Fragment>
            )}
          </motion.div>
        </AnimatePresence>
      </div>
    </div>
  );
};

export default SocialLinkHeader;

```

## File: connectfy-client/src/modules/users/UserProfile/ui/components/SocialLinks/components/SocialPlatformIcons.tsx
```typescript
import { Mail, Globe, Link } from "lucide-react";
import InstagramIcon from "@/assets/icons/InstagramIcon";
import FacebookIcon from "@/assets/icons/FacebookIcon";
import XIcon from "@/assets/icons/XIcon";
import LinkedInIcon from "@/assets/icons/LinkedInIcon";
import YoutubeIcon from "@/assets/icons/YoutubeIcon";
import GithubIcon from "@/assets/icons/GithubIcon";
import RedditIcon from "@/assets/icons/RedditIcon";
import PinterestIcon from "@/assets/icons/PinterestIcon";
import TiktokIcon from "@/assets/icons/TiktokIcon";
import TelegramIcon from "@/assets/icons/TelegramIcon";
import DiscordIcon from "@/assets/icons/DiscordIcon";
import SnapchatIcon from "@/assets/icons/SnapchatIcon";
import WhatsappIcon from "@/assets/icons/WhatsappIcon";
import MediumIcon from "@/assets/icons/MediumIcon";
import BehanceIcon from "@/assets/icons/BehanceIcon";
import { SOCIAL_LINK_PLATFORM } from "@/common/enums/enums";

export const PLATFORM_ICONS: Record<SOCIAL_LINK_PLATFORM, React.ReactNode> = {
  [SOCIAL_LINK_PLATFORM.INSTAGRAM]: <InstagramIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.FACEBOOK]: <FacebookIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.X]: <XIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.LINKEDIN]: <LinkedInIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.YOUTUBE]: <YoutubeIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.GITHUB]: <GithubIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.REDDIT]: <RedditIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.PINTEREST]: <PinterestIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.TIKTOK]: <TiktokIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.TELEGRAM]: <TelegramIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.DISCORD]: <DiscordIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.SNAPCHAT]: <SnapchatIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.WHATSAPP]: <WhatsappIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.MEDIUM]: <MediumIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.BEHANCE]: <BehanceIcon className="w-4.5 h-4.5" />,
  [SOCIAL_LINK_PLATFORM.EMAIL]: (
    <Mail size={18} className="text-(--text-color)!" />
  ),
  [SOCIAL_LINK_PLATFORM.WEBSITE]: (
    <Globe size={18} className="text-(--text-color)!" />
  ),
  [SOCIAL_LINK_PLATFORM.OTHER]: (
    <Link size={18} className="text-(--text-color)!" />
  ),
};

```

## File: connectfy-client/src/modules/users/UserProfile/ui/components/SocialLinks/components/SocialContextMenu.tsx
```typescript
import { FC } from "react";
import { motion } from "framer-motion";
import { Copy, ExternalLink } from "lucide-react";
import { useTranslation } from "react-i18next";
import ContextMenuItem from "@/components/ContextMenu/ContextMenuItem";

interface ContextMenuProps {
  onCopy: () => void;
  onOpenLink: () => void;
}

const SocialContextMenu: FC<ContextMenuProps> = ({ onCopy, onOpenLink }) => {
  const { t } = useTranslation();

  return (
    <motion.div
      initial={{ opacity: 0, scale: 0.9, y: -10 }}
      animate={{ opacity: 1, scale: 1, y: 0 }}
      exit={{ opacity: 0, scale: 0.9, y: -10 }}
      transition={{ duration: 0.15, ease: "easeOut" }}
      className="flex flex-col min-w-[220px] bg-(--bg-color) rounded-2xl shadow-2xl overflow-hidden"
    >
      <ContextMenuItem
        icon={ExternalLink}
        label={t("common.open")}
        onClick={onOpenLink}
      />
      <ContextMenuItem icon={Copy} label={t("common.copy")} onClick={onCopy} />
    </motion.div>
  );
};

export default SocialContextMenu;

```

## File: connectfy-client/src/modules/users/UserProfile/router/router.tsx
```typescript
import { RouteObject } from "react-router-dom";
import { lazy } from "react";
import { ROUTER } from "@/common/constants/routet";
import ComponentLoader from "@/components/Loader/Components/ComponentLoader";
import MainCardSkeleton from "@/components/Skeleton/profile/MainCardSkeleton";
import PersonalInformationSkeleton from "@/components/Skeleton/profile/PersonalInformationSkeleton";
import BioSkeleton from "@/components/Skeleton/profile/BioSkeleton";
import ProfileHeader from "../ui/components/ProfileHeader/ProfileHeader";

const ProfilePageSkeleton = () => (
  <div className="relative w-full h-screen overflow-x-hidden overflow-y-auto font-sans scroll-smooth bg-(--bg-color)">
    <ProfileHeader />

    <main className="mx-auto max-w-[900px] pt-8 px-6 pb-[60px]">
      <MainCardSkeleton />
      <PersonalInformationSkeleton />
      <BioSkeleton />
    </main>
  </div>
);

const Profile = ComponentLoader(
  lazy(() => import("../ui/Profile")),
  <ProfilePageSkeleton />,
);

const routes: RouteObject[] = [
  {
    path: `${ROUTER.USERS.PROFILE}/:id`,
    element: <Profile />,
  },
];

export default routes;

```

## File: connectfy-client/src/modules/users/UserProfile/types/types.ts
```typescript
import {
  FriendshipStatus,
  GENDER,
  SOCIAL_LINK_PLATFORM,
} from "@/common/enums/enums";
import { IPhoneNumber } from "@/modules/auth/types/types";
import { IAvatar } from "@/modules/profile/types/types";

export interface IFindOneUserResponse {
  user: IFindOneUser;
  profile: IFindOneProfile;
  relationship: IFindOneRelationship;
  actions: {
    canSendFriendRequest: boolean;
    canSendMessage: boolean;
  };
}

export interface IFindOneUser {
  _id: string;
  username: string;
  email: string | null;
  phoneNumber: IPhoneNumber | null;
  createdAt: Date;
}

export interface IFindOneProfile {
  _id: string;
  firstName: string;
  lastName: string;
  fullName: string;
  gender: GENDER | null;
  bio: string | null;
  location: string | null;
  avatar: IAvatar | null;
  birthdayDate: Date | null;
  lastSeen: Date;
}

export interface IFindOneRelationship {
  friendship: {
    _id: string;
    status: FriendshipStatus;
    userId: string;
    isFavorite: boolean;
    isMuted: boolean;
  } | null;
  count: number;
}

export interface IFindOneActions {
  canSendFriendRequest: boolean;
  canSendMessage: boolean;
  isBlocked: boolean;
  hasBlocked: boolean;
}

export interface IFindSocialLinkResponse {
  _id: string;
  userId: string;
  name: string;
  rank: number;
  url: string;
  platform: SOCIAL_LINK_PLATFORM;
  createdAt: Date;
  updatedAt: Date;
}

```

## File: connectfy-client/src/modules/users/UserProfile/api/api.ts
```typescript
import { baseQuery } from "@/common/api/axiosBaseQuery";
import { RESOURCE } from "@/common/enums/enums";
import { createApi } from "@reduxjs/toolkit/query/react";
import { API_ENDPOINTS } from "@/common/constants/apiEndpoints";
import { IFindOneUserResponse } from "../types/types";

export const userProfileApi = createApi({
  reducerPath: RESOURCE.USER_PROFİLE,
  baseQuery: baseQuery,
  tagTypes: ["UserProfile"],
  endpoints: (builder) => ({
    findUser: builder.query<IFindOneUserResponse, string>({
      query: (userId: string) => ({
        url: API_ENDPOINTS.ACCOUNT.PROFILE.FIND_ONE(userId),
        method: "POST",
      }),
      providesTags: (_, __, userId) => [{ type: "UserProfile", id: userId }],
    }),
  }),
});

export const { useFindUserQuery } = userProfileApi;

```

## File: connectfy-client/src/modules/users/AllUsers/ui/AllUsers.tsx
```typescript
import { useTranslation } from "react-i18next";
import { Fragment, useState, useCallback, useEffect } from "react";
import { Search, X } from "lucide-react";
import { useSearchParams } from "react-router-dom";

import UniqueHeader from "@/components/Header/UnqiueHeader/UniqueHeader";
import { LoadingSpinner } from "@/components/Spinner/Settings/LoadingSpinner";
import Input from "@/components/ui/CustomInput/Input/Input";
import Button from "@/components/ui/CustomButton/Button/Button";
import UserCard from "@/components/Card/UserCard/UserCard";

import { useAppNavigation } from "@/hooks/useAppNavigation";
import { ROUTER } from "@/common/constants/routet";
import { useSearchQuery } from "../api/api";
import { useUser } from "@/context/UserContext";
import { checkEmptyString } from "@/common/utils/checkValues";
import { useFriendship } from "../../MyFriends/hooks/useFriendship";
import FriendshipActions from "./components/FriendshipActions";
import { AnimatePresence, motion } from "framer-motion";
import NoProfilePhotoIcon from "@/assets/icons/NoProfilePhotoIcon";

const AllUsers = () => {
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();
  const { user: currentUser } = useUser();
  const {
    handleSendFriendRequest,
    handleAcceptFriendRequest,
    handleDeclineFriendRequest,
    handleCancelFriendRequest,
    handleRemoveFriendship,
    isActionLoading,
  } = useFriendship();

  const [urlParams, setUrlParams] = useSearchParams();
  const queryParam = urlParams.get("q") || "";

  const [inputValue, setInputValue] = useState(queryParam);
  const [lastAcceptedUser, setLastAcceptedUser] = useState<{
    fullName: string;
    avatar: string | null;
  } | null>(null);

  const { data, isLoading, isFetching } = useSearchQuery(
    { search: queryParam, skip: 1, limit: 10 },
    {
      skip: !currentUser?._id || !checkEmptyString(queryParam),
      refetchOnMountOrArgChange: true,
    },
  );

  const executeSearch = useCallback(
    (value: string) => {
      const trimmed = value.trim();
      if (trimmed) {
        setUrlParams({ q: trimmed }, { replace: true });
      } else {
        setUrlParams({}, { replace: true });
      }
    },
    [setUrlParams],
  );

  const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    if (e.key === "Enter") executeSearch(inputValue);
  };

  useEffect(() => {
    setInputValue(queryParam);
  }, [queryParam]);

  useEffect(() => {
    if (lastAcceptedUser) {
      const timer = setTimeout(() => setLastAcceptedUser(null), 5000);
      return () => clearTimeout(timer);
    }
  }, [lastAcceptedUser]);

  const users = data?.data ?? [];

  return (
    <Fragment>
      <section className="w-full h-full pb-5 flex flex-col box-border bg-(--bg-color) text-(--text-color)">
        <UniqueHeader
          onClickBack={() => navigate(ROUTER.USERS.MAIN)}
          headerTitle={t("common.find_user")}
          headerSubtitle={t("common.find_user_description")}
        />

        {/* ── Search Bar ── */}
        <div className="px-4 pt-1 pb-3">
          <div className="relative flex items-center gap-2">
            <Input
              id="user-search"
              type="search"
              inputSize="large"
              icon={<Search size={16} />}
              title={t("common.search")}
              value={inputValue}
              onChange={(e) => setInputValue(e.target.value)}
              onKeyDown={handleKeyDown}
              className="block w-full ps-9 bg-neutral-secondary-medium border border-default-medium text-heading text-sm rounded-base focus:ring-brand focus:border-brand shadow-xs"
            />
            <Button
              type="button"
              onClick={() => executeSearch(inputValue)}
              icon={<Search size={15} />}
              className="flex bg-(--btn-edit-bg)! text-(--btn-edit-text)! font-semibold p-5 rounded-lg items-center justify-center transition-all hover:opacity-80"
              disabled={!checkEmptyString(inputValue)}
            />
          </div>
        </div>

        <AnimatePresence>
          {lastAcceptedUser && (
            <motion.div
              initial={{ height: 0, opacity: 0, marginBottom: 0 }}
              animate={{ height: "auto", opacity: 1, marginBottom: 16 }}
              exit={{ height: 0, opacity: 0, marginBottom: 0 }}
              className="overflow-hidden px-4"
            >
              <div className="flex items-center justify-between p-4 bg-(--icon-green-bg) border border-(--icon-green-text) border-opacity-20 rounded-2xl shadow-sm">
                <div className="flex items-center gap-3">
                  {lastAcceptedUser.avatar ? (
                    <img
                      src={lastAcceptedUser.avatar}
                      alt={lastAcceptedUser.fullName}
                      className="w-10 h-10 rounded-full object-cover"
                      loading="eager"
                      fetchPriority="high"
                      decoding="async"
                    />
                  ) : (
                    <div className="flex items-center justify-center w-full h-full bg-(--active-bg-2)">
                      <NoProfilePhotoIcon />
                    </div>
                  )}
                  <p className="text-sm font-medium text-(--icon-green-text)">
                    {t("common.now_friends_with", {
                      name: lastAcceptedUser.fullName,
                    })}
                  </p>
                </div>
                <Button
                  onClick={() => setLastAcceptedUser(null)}
                  className="text-(--icon-green-text) opacity-60 hover:opacity-100 transition-opacity"
                  icon={<X size={18} />}
                />
              </div>
            </motion.div>
          )}
        </AnimatePresence>

        {/* ── Results ── */}
        <div className="flex-1 min-h-0 overflow-y-auto px-3">
          {isLoading || isFetching ? (
            <LoadingSpinner />
          ) : (
            <motion.ul
              initial={{ opacity: 0 }}
              animate={{ opacity: 1 }}
              className="space-y-1"
            >
              {users.length > 0 ? (
                users.map((u) => (
                  <UserCard
                    key={u._id}
                    user={u}
                    actionButtons={
                      <FriendshipActions
                        user={u}
                        currentUserId={currentUser?._id ?? ""}
                        onSend={(id) =>
                          handleSendFriendRequest({ friendId: id })
                        }
                        onAccept={async (data) => {
                          await handleAcceptFriendRequest(data);
                          setLastAcceptedUser({
                            fullName: u.fullName || `${u.firstName} ${u.lastName}`,
                            avatar: u.avatar?.url || null,
                          });
                        }}
                        onDecline={(data) => {
                          handleDeclineFriendRequest(data);
                        }}
                        onCancel={(data) => {
                          handleCancelFriendRequest(data);
                        }}
                        onRemove={(data) => {
                          handleRemoveFriendship(data);
                        }}
                        loading={isActionLoading}
                      />
                    }
                  />
                ))
              ) : (
                <div className="flex flex-col items-center justify-center py-16 gap-3 text-(--muted-color)">
                  <Search size={32} strokeWidth={1.5} className="opacity-40" />
                  <p className="text-sm font-medium">
                    {queryParam
                      ? t("common.no_results_found", { query: queryParam })
                      : t("common.start_searching")}
                  </p>
                </div>
              )}
            </motion.ul>
          )}
        </div>
      </section>
    </Fragment>
  );
};

export default AllUsers;

```

## File: connectfy-client/src/modules/users/AllUsers/ui/components/FriendshipActions.tsx
```typescript
import Button from "@/components/ui/CustomButton/Button/Button";
import { ISearchUserResult } from "../../types/types";
import { Check, Clock, UserCheck, UserPlus, X } from "lucide-react";
import { FriendshipStatus } from "@/common/enums/enums";
import { FC } from "react";
import {
  IAcceptFriendshipRequest,
  ICancelFriendshipRequest,
  IDeclineFriendshipRequest,
  IRemoveFriendship,
} from "@/modules/users/MyFriends/types/types";

interface IProps {
  user: ISearchUserResult;
  currentUserId: string;
  onSend: (id: string) => void;
  onAccept: (data: IAcceptFriendshipRequest) => void;
  onDecline: (data: IDeclineFriendshipRequest) => void;
  onCancel: (data: ICancelFriendshipRequest) => void;
  onRemove: (data: IRemoveFriendship) => void;
  loading: boolean;
}

const FriendshipActions: FC<IProps> = ({
  user,
  currentUserId,
  onSend,
  onAccept,
  onDecline,
  onCancel,
  onRemove,
  loading,
}) => {
  const { _id, relationship } = user;

  // 1. Heç bir münasibət yoxdursa -> İSTƏK GÖNDƏR
  if (!relationship) {
    return (
      <Button
        onClick={() => onSend(_id)}
        className="p-2 rounded-lg transition-all border border-(--active-bg)"
        disabled={loading}
        isLoading={loading}
        icon={<UserPlus size={15} />}
      />
    );
  }

  const { status, userId: requesterId } = relationship;
  const isAccepted = status === FriendshipStatus.Accepted;
  const isPending = status === FriendshipStatus.Pending;
  // Bizə gələn istəkdirmi? (İstəyi atan mən deyiləmsə və status pendingdirsə)
  const isIncomingRequest = isPending && requesterId !== currentUserId;

  // 2. Dostdurlarsa -> USER CHECK (Dostluq təsdiqlənib)
  if (isAccepted) {
    return (
      <Button
        className="p-2 rounded-lg bg-(--active-bg) text-(--primary-color) border border-(--primary-color)"
        icon={<UserCheck size={15} />}
        disabled={loading}
        isLoading={loading}
        onClick={() =>
          onRemove({ friendshipId: relationship._id, userId: _id })
        }
      />
    );
  }

  // 3. Status PENDING-dirsə
  if (isPending) {
    // 3a. Əgər mən göndərmişəmsə -> CLOCK (Gözləmədə)
    if (!isIncomingRequest) {
      return (
        <Button
          className="p-2 rounded-lg border border-(--active-bg) opacity-70"
          icon={<Clock size={15} />}
          disabled={loading}
          isLoading={loading}
          onClick={() =>
            onCancel({ friendshipId: relationship._id, userId: _id })
          }
        />
      );
    }

    // 3b. Əgər mənə gəlibsə -> ACCEPT & DECLINE (Qəbul et / Rədd et)
    return (
      <div className="flex items-center gap-2">
        <Button
          onClick={() =>
            onAccept({ friendshipId: relationship._id, userId: _id })
          }
          className="p-2 rounded-lg bg-green-500/10 text-green-500 border border-green-500/50"
          disabled={loading}
          icon={<Check size={15} />}
          isLoading={loading}
        />
        <Button
          onClick={() =>
            onDecline({ friendshipId: relationship._id, userId: _id })
          }
          className="p-2 rounded-lg bg-red-500/10 text-red-500 border border-red-500/50"
          disabled={loading}
          icon={<X size={15} />}
          isLoading={loading}
        />
      </div>
    );
  }

  return null;
};

export default FriendshipActions;

```

## File: connectfy-client/src/modules/users/AllUsers/router/router.tsx
```typescript
import { RouteObject } from "react-router-dom";
import { lazy } from "react";
import { ROUTER } from "@/common/constants/routet";
import ComponentLoader from "@/components/Loader/Components/ComponentLoader";

const AllUsers = ComponentLoader(lazy(() => import("../ui/AllUsers")));

const routes: RouteObject[] = [
  {
    path: ROUTER.USERS.SEARCH,
    element: <AllUsers />,
  },
];

export default routes;

```

## File: connectfy-client/src/modules/users/AllUsers/types/types.ts
```typescript
import { FriendshipStatus } from "@/common/enums/enums";
import { IAvatar } from "@/modules/profile/types/types";

export interface ISearchUserResult {
  _id: string;
  username: string;
  firstName: string;
  lastName: string;
  fullName: string;
  avatar: IAvatar | null;
  relationship: {
    _id: string;
    status: FriendshipStatus;
    userId: string;
    friendId: string;
  } | null;
  friendshipRequest: boolean;
}

export interface ISearchUsers {
  search: string;
  limit: number;
  skip: number;
}

```

## File: connectfy-client/src/modules/users/AllUsers/api/api.ts
```typescript
import { baseQuery } from "@/common/api/axiosBaseQuery";
import { RESOURCE } from "@/common/enums/enums";
import { createApi } from "@reduxjs/toolkit/query/react";
import { API_ENDPOINTS } from "@/common/constants/apiEndpoints";
import { IFindAllResponse } from "@/common/interfaces/interfaces";
import { ISearchUserResult, ISearchUsers } from "../types/types";

export const allUsersApi = createApi({
  reducerPath: RESOURCE.ALL_USERS,
  baseQuery: baseQuery,
  tagTypes: ["AllUsers"],
  endpoints: (builder) => ({
    search: builder.query<IFindAllResponse<ISearchUserResult>, ISearchUsers>({
      query: (body) => ({
        url: API_ENDPOINTS.ACCOUNT.PROFILE.SEARCH,
        method: "POST",
        body,
      }),
      providesTags: ["AllUsers"],
    }),
  }),
});

export const { useSearchQuery } = allUsersApi;

```

## File: connectfy-client/src/modules/users/Users/ui/Users.tsx
```typescript
import { useTranslation } from "react-i18next";
import {
  Users as UsersIcon,
  UserPlus,
  UserSearch,
  UserRoundX,
} from "lucide-react";
import { ROUTER } from "@/common/constants/routet";
import { memo } from "react";
import { useAppNavigation } from "@/hooks/useAppNavigation";

const Users = () => {
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();

  const usersOptions = [
    {
      title: t("common.find_user"),
      description: t("common.find_user_description"),
      icon: <UserSearch size={24} />,
      gradient: "from-blue-500 to-blue-600",
      path: ROUTER.USERS.SEARCH,
    },
    {
      title: t("common.friends"),
      description: t("common.friends_description"),
      icon: <UsersIcon size={24} />,
      gradient: "from-pink-500 to-pink-600",
      path: ROUTER.USERS.FRIENDS,
    },
    {
      title: t("common.friendship_requests"),
      description: t("common.friendship_requests_description"),
      icon: <UserPlus size={24} />,
      gradient: "from-violet-500 to-purple-600",
      path: ROUTER.USERS.REQUESTS,
    },
    {
      title: t("common.blocked_users"),
      description: t("common.blocked_users_description"),
      icon: <UserRoundX size={24} />,
      gradient: "from-sky-400 to-blue-600",
      path: ROUTER.USERS.BLOCKLIST,
    },
  ];

  return (
    <div className="w-full min-h-screen p-6 transition-colors duration-300 bg-(--bg-color)">
      <div className="max-w-[900px] mx-auto">
        {/* Header / Hero Section */}
        <div className="text-center mb-12 animate-in fade-in slide-in-from-top-4 duration-700">
          <div className="w-24 h-24 mx-auto mb-6 flex items-center justify-center text-white rounded-[24px] bg-(--primary-color) shadow-(--active-shadow)">
            <UsersIcon size={48} strokeWidth={2} />
          </div>
          <h1 className="text-4xl font-extrabold mb-3 text-(--text-primary)">
            {t("common.users_title")}
          </h1>
          <p className="max-w-[600px] mx-auto text-lg leading-relaxed text-(--muted-color)">
            {t("common.users_description")}
          </p>
        </div>

        {/* Users Grid */}
        <div className="grid grid-cols-1 md:grid-cols-2 gap-5 mb-8">
          {usersOptions.map((option, index) => (
            <div
              key={index}
              role="button"
              tabIndex={0}
              onClick={() => navigate(option.path)}
              className="group flex items-center gap-4 p-6 rounded-2xl cursor-pointer transition-all duration-300 border border-transparent 
                         bg-(--card-bg) shadow-(--card-shadow) 
                         hover:shadow-xl hover:-translate-y-1 hover:border-(--primary-color)
                         active:scale-[0.98]"
            >
              {/* Icon Container */}
              <div
                className={`w-14 h-14 shrink-0 rounded-xl flex items-center justify-center text-white bg-linear-to-br ${option.gradient} shadow-inner`}
              >
                {option.icon}
              </div>

              {/* Text Info */}
              <div className="text-left">
                <h3 className="text-lg font-bold mb-1 transition-colors text-(--text-primary) group-hover:text-(--primary-color)">
                  {option.title}
                </h3>
                <p className="text-sm leading-snug text-(--muted-color)">
                  {option.description}
                </p>
              </div>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};

export default memo(Users);

```

## File: connectfy-client/src/modules/users/Users/router/router.tsx
```typescript
import { RouteObject } from "react-router-dom";
import { lazy } from "react";
import { ROUTER } from "@/common/constants/routet";
import ComponentLoader from "@/components/Loader/Components/ComponentLoader";

const Users = ComponentLoader(lazy(() => import("../ui/Users")));

const routes: RouteObject[] = [
  {
    path: ROUTER.USERS.MAIN,
    element: <Users />,
  },
];

export default routes;

```

## File: connectfy-client/src/modules/users/MyFriends/hooks/useFriendship.tsx
```typescript
import { useErrors } from "@/hooks/useErrors";
import {
  useAcceptFriendRequestMutation,
  useCancelFriendRequestMutation,
  useDeclineFriendRequestMutation,
  useRemoveFriendshipMutation,
  useSendFriendRequestMutation,
  useUpdateCloseFriendMutation,
  useUpdateNotificationMutation,
} from "../api/api";
import {
  IAcceptFriendshipRequest,
  ICancelFriendshipRequest,
  IDeclineFriendshipRequest,
  IRemoveFriendship,
  ISendFriendshipRequest,
  IUpdateCloseFriend,
  IUpdateNotification,
} from "../types/types";
import { useUser } from "@/context/UserContext";
import friendshipAudio from "@/assets/audio/friendship-audio.mp3";

export const useFriendship = () => {
  const { user } = useUser();
  const { showResponseErrors } = useErrors();

  const [sendFriendRequest, { isLoading: isSendFriendRequestLoading }] =
    useSendFriendRequestMutation();
  const [acceptFriendRequest, { isLoading: isAcceptFriendRequestLoading }] =
    useAcceptFriendRequestMutation();
  const [declineFriendRequest, { isLoading: isDeclineFriendRequestLoading }] =
    useDeclineFriendRequestMutation();
  const [removeFriendship, { isLoading: isRemoveFriendshipLoading }] =
    useRemoveFriendshipMutation();
  const [cancelFriendRequest, { isLoading: isCancelFriendRequestLoading }] =
    useCancelFriendRequestMutation();
  const [updateCloseFriend, { isLoading: isUpdateCloseFriendLoading }] =
    useUpdateCloseFriendMutation();
  const [updateNotification, { isLoading: isUpdateNotificationLoading }] =
    useUpdateNotificationMutation();

  const handleSyncError = (error: any) => {
    const isSilent = !!(
      error?.additional?.relationship || error?.additional?.alreadyCancelled
    );
    if (!isSilent) showResponseErrors(error);
  };

  const playAudio = () => {
    const audio = new Audio(friendshipAudio);
    audio.volume = 0.35;
    audio.play().catch(() => {});
  };

  const handleSendFriendRequest = async (
    data: Omit<ISendFriendshipRequest, "userId">,
  ) => {
    try {
      const payload = { userId: user?._id || "", friendId: data.friendId };
      const response = await sendFriendRequest(payload).unwrap();
      playAudio();
      return response;
    } catch (error) {
      handleSyncError(error);
    }
  };

  const handleAcceptFriendRequest = async (data: IAcceptFriendshipRequest) => {
    try {
      await acceptFriendRequest(data).unwrap();
      playAudio();
    } catch (error) {
      handleSyncError(error);
    }
  };

  const handleDeclineFriendRequest = async (
    data: IDeclineFriendshipRequest,
  ) => {
    try {
      await declineFriendRequest(data).unwrap();
    } catch (error) {
      handleSyncError(error);
    }
  };

  const handleRemoveFriendship = async (data: IRemoveFriendship) => {
    try {
      await removeFriendship(data).unwrap();
    } catch (error) {
      handleSyncError(error);
    }
  };

  const handleCancelFriendRequest = async (data: ICancelFriendshipRequest) => {
    try {
      const response = await cancelFriendRequest(data).unwrap();
      return response;
    } catch (error) {
      handleSyncError(error);
    }
  };

  const handleUpdateCloseFriend = async (data: IUpdateCloseFriend) => {
    try {
      await updateCloseFriend(data).unwrap();
    } catch (error) {
      showResponseErrors(error);
    }
  };

  const handleUpdateNotification = async (data: IUpdateNotification) => {
    try {
      await updateNotification(data).unwrap();
    } catch (error) {
      showResponseErrors(error);
    }
  };

  return {
    handleSendFriendRequest,
    handleAcceptFriendRequest,
    handleDeclineFriendRequest,
    handleRemoveFriendship,
    handleCancelFriendRequest,
    handleUpdateCloseFriend,
    handleUpdateNotification,

    isActionLoading:
      isSendFriendRequestLoading ||
      isAcceptFriendRequestLoading ||
      isDeclineFriendRequestLoading ||
      isRemoveFriendshipLoading ||
      isCancelFriendRequestLoading,

    isUpdateCloseFriendLoading,
    isUpdateNotificationLoading,
  };
};

```

## File: connectfy-client/src/modules/users/MyFriends/ui/MyFriends.tsx
```typescript
import { useUser } from "@/context/UserContext";
import { useFindFriendsQuery } from "../api/api";
import { useTranslation } from "react-i18next";
import { useAppNavigation } from "@/hooks/useAppNavigation";
import { ROUTER } from "@/common/constants/routet";
import UniqueHeader from "@/components/Header/UnqiueHeader/UniqueHeader";
import Input from "@/components/ui/CustomInput/Input/Input";
import { Search, Star, VolumeX, Users } from "lucide-react";
import { useSearchParams } from "react-router-dom";
import { useCallback, useEffect, useState } from "react";
import Button from "@/components/ui/CustomButton/Button/Button";
import { checkEmptyString } from "@/common/utils/checkValues";
import { LoadingSpinner } from "@/components/Spinner/Settings/LoadingSpinner";
import UserCard from "@/components/Card/UserCard/UserCard";
import { FriendshipStatus } from "@/common/enums/enums";
import { useFriendship } from "../hooks/useFriendship";
import FriendshipActions from "./components/FriendshipActions";
import { IFriendFilters } from "../types/types";
import { motion } from "framer-motion"; // <-- Framer Motion əlavə edildi
import { useIsMobile } from "@/hooks/useIsMobile";

const MyFriends = () => {
  const { user: currentUser } = useUser();
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();
  const isMobile = useIsMobile();

  const [urlParams, setUrlParams] = useSearchParams();
  const queryParam = urlParams.get("q") || "";

  const { handleRemoveFriendship, isActionLoading } = useFriendship();

  const [inputValue, setInputValue] = useState(queryParam);

  const [filters, setFilters] = useState<IFriendFilters>({
    favorite: false,
    muted: false,
  });

  const isAll = !filters.favorite && !filters.muted;

  const { data, isLoading, isFetching } = useFindFriendsQuery(
    {
      userId: currentUser?._id || "",
      skip: 1,
      limit: 10,
      search: queryParam,
      isFavorite: filters.favorite,
      isMuted: filters.muted,
    },
    {
      skip: !currentUser?._id,
      refetchOnMountOrArgChange: true,
    },
  );

  const executeSearch = useCallback(
    (value: string) => {
      const trimmed = value.trim();
      if (trimmed) {
        setUrlParams({ q: trimmed }, { replace: true });
      } else {
        setUrlParams({}, { replace: true });
      }
    },
    [setUrlParams],
  );

  const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    if (e.key === "Enter") executeSearch(inputValue);
  };

  useEffect(() => {
    setInputValue(queryParam);
  }, [queryParam]);

  const handleFilterToggle = useCallback(
    (type: "all" | "favorite" | "muted") => {
      if (type === "all") {
        setFilters({ favorite: false, muted: false });
      } else {
        setFilters((prev) => ({
          ...prev,
          [type]: !prev[type],
        }));
      }
    },
    [],
  );

  const users = data?.data ?? [];

  // Motion Button üçün ortaq class-lar (Daha premium görünüş)
  const baseFilterBtnClass =
    "relative flex items-center justify-center gap-2 px-4 py-2 text-sm font-medium rounded-lg transition-all duration-300 focus:outline-none focus-visible:ring-2 focus-visible:ring-(--primary-color) select-none cursor-pointer w-full";

  return (
    <section className="w-full h-full pb-5 flex flex-col box-border bg-(--bg-color) text-(--text-color)">
      <UniqueHeader
        onClickBack={() => navigate(ROUTER.USERS.MAIN)}
        headerTitle={t("common.my_friends")}
        headerSubtitle={t("common.my_friends_description")}
      />

      {/* ── Search Bar ── */}
      <div className="px-4 pt-1 pb-3">
        <div className="relative flex items-center gap-2">
          <Input
            id="user-search"
            type="search"
            inputSize="large"
            icon={<Search size={16} />}
            title={t("common.search")}
            value={inputValue}
            onChange={(e) => setInputValue(e.target.value)}
            onKeyDown={handleKeyDown}
            className="block w-full ps-9 bg-(--input-bg) border border-(--input-border) text-(--text-primary) text-sm rounded-xl focus:ring-(--primary-color) focus:border-(--primary-color) shadow-sm transition-colors"
          />
          <Button
            type="button"
            onClick={() => executeSearch(inputValue)}
            icon={<Search size={15} />}
            className="flex bg-(--btn-edit-bg) text-(--btn-edit-text) font-semibold p-5 rounded-xl items-center justify-center transition-all hover:scale-105 active:scale-95"
            disabled={!checkEmptyString(inputValue)}
          />
        </div>

        {/* ── Motion Group Filters (Orta, Segmented UI) ── */}
        <div className="flex justify-center mt-5 w-full">
          {/* Konteyner: Yumşaq arxa fon və padding */}
          <div
            className="inline-flex w-full items-center p-1 bg-(--input-bg) border border-(--auth-glass-border) rounded-xl shadow-xs gap-1"
            role="group"
          >
            {/* ALL Düyməsi */}
            <motion.button
              whileTap={{ scale: 0.95 }}
              onClick={() => handleFilterToggle("all")}
              className={`
                ${baseFilterBtnClass}
                ${
                  isAll
                    ? "bg-(--primary-color) text-white shadow-md"
                    : "bg-transparent text-(--muted-color) hover:text-(--text-primary) hover:bg-(--active-bg-2)"
                }
              `}
            >
              <Users size={16} />
              {isMobile ? "" : t("common.all")}
            </motion.button>

            {/* FAVOURITES Düyməsi */}
            <motion.button
              whileTap={{ scale: 0.95 }}
              onClick={() => handleFilterToggle("favorite")}
              className={`
                ${baseFilterBtnClass}
                ${
                  filters.favorite
                    ? "bg-(--primary-color) text-white shadow-md"
                    : "bg-transparent text-(--muted-color) hover:text-(--text-primary) hover:bg-(--active-bg-2)"
                }
              `}
            >
              <motion.div
                animate={{
                  scale: filters.favorite ? [1, 1.2, 1] : 1,
                }}
                transition={{ duration: 0.3 }}
                className="flex items-center justify-center"
              >
                <Star
                  size={16}
                  fill={filters.favorite ? "currentColor" : "none"}
                />
              </motion.div>
              {isMobile ? "" : t("common.close_friend")}
            </motion.button>

            {/* MUTED Düyməsi */}
            <motion.button
              whileTap={{ scale: 0.95 }}
              onClick={() => handleFilterToggle("muted")}
              className={`
                ${baseFilterBtnClass}
                ${
                  filters.muted
                    ? "bg-(--primary-color) text-white shadow-md"
                    : "bg-transparent text-(--muted-color) hover:text-(--text-primary) hover:bg-(--active-bg-2)"
                }
              `}
            >
              <motion.div
                animate={{
                  rotate: filters.muted ? [0, -15, 15, 0] : 0,
                }}
                transition={{ duration: 0.4 }}
                className="flex items-center justify-center"
              >
                <VolumeX size={16} />
              </motion.div>
              {isMobile ? "" : t("common.muted")}
            </motion.button>
          </div>
        </div>
      </div>

      {/* ── Results ── */}
      <div className="flex-1 min-h-0 overflow-y-auto px-3">
        {isLoading || isFetching ? (
          <LoadingSpinner />
        ) : (
          <motion.ul
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            className="space-y-1"
          >
            {users.length > 0 ? (
              users.map((u, index) => (
                <motion.div
                  key={u._id}
                  initial={{ opacity: 0, y: 10 }}
                  animate={{ opacity: 1, y: 0 }}
                  transition={{ delay: index * 0.05 }} // List elementləri üçün kiçik fade-in animasiyası
                >
                  <UserCard
                    user={{
                      ...u.friendId,
                      avatar: {
                        key: null,
                        url: u.friendId.avatar,
                        isCustom: false,
                      },
                      relationship: {
                        status: FriendshipStatus.Accepted,
                        userId: (u.userId as Record<string, any>)._id,
                        friendId: u.friendId._id,
                        _id: u._id,
                      },
                      friendshipRequest: false,
                    }}
                    actionButtons={
                      <FriendshipActions
                        relationship={u}
                        onRemove={(data) => {
                          handleRemoveFriendship(data);
                        }}
                        loading={isActionLoading}
                      />
                    }
                  />
                </motion.div>
              ))
            ) : (
              <motion.div
                initial={{ opacity: 0, scale: 0.9 }}
                animate={{ opacity: 1, scale: 1 }}
                className="flex flex-col items-center justify-center h-48 gap-3 text-(--muted-color)"
              >
                <Search size={40} strokeWidth={1} className="opacity-30" />
                <p className="text-sm font-medium">
                  {queryParam
                    ? t("common.no_results_found", { query: queryParam })
                    : t("common.start_searching")}
                </p>
              </motion.div>
            )}
          </motion.ul>
        )}
      </div>
    </section>
  );
};

export default MyFriends;

```

## File: connectfy-client/src/modules/users/MyFriends/ui/components/FriendshipActions.tsx
```typescript
import Button from "@/components/ui/CustomButton/Button/Button";
import { UserCheck } from "lucide-react";
import { FriendshipStatus } from "@/common/enums/enums";
import { FC } from "react";
import {
  IFriendship,
  IRemoveFriendship,
} from "@/modules/users/MyFriends/types/types";

interface IProps {
  relationship: IFriendship;
  onRemove: (data: IRemoveFriendship) => void;
  loading: boolean;
}

const FriendshipActions: FC<IProps> = ({ relationship, onRemove, loading }) => {
  const { _id, userId } = relationship;

  if (relationship?.status !== FriendshipStatus.Accepted) {
    return null;
  }

  return (
    <Button
      className="p-2 rounded-lg bg-(--active-bg) text-(--primary-color) border border-(--primary-color)"
      icon={<UserCheck size={15} />}
      disabled={loading}
      isLoading={loading}
      onClick={() => onRemove({ friendshipId: _id, userId: userId as string })}
    />
  );
};

export default FriendshipActions;

```

## File: connectfy-client/src/modules/users/MyFriends/router/router.tsx
```typescript
import { RouteObject } from "react-router-dom";
import { lazy } from "react";
import { ROUTER } from "@/common/constants/routet";
import ComponentLoader from "@/components/Loader/Components/ComponentLoader";

const MyFriends = ComponentLoader(lazy(() => import("../ui/MyFriends")));

const routes: RouteObject[] = [
  {
    path: ROUTER.USERS.FRIENDS,
    element: <MyFriends />,
  },
];

export default routes;

```

## File: connectfy-client/src/modules/users/MyFriends/types/types.ts
```typescript
import { FriendshipRequestType, FriendshipStatus } from "@/common/enums/enums";

export interface IFriendFilters {
  favorite: boolean;
  muted: boolean;
}

export interface IFriendship {
  _id: string;
  userId: string | Record<string, any>;
  friendId: {
    _id: string;
    firstName: string;
    lastName: string;
    fullName: string;
    username: string;
    avatar: string;
  };
  status: FriendshipStatus;
  isFavorite: boolean;
  isMuted: boolean;
  createdAt: Date;
  updatedAt: Date;
}

export interface IReturnedFriendship {
  _id: string;
  userId: string;
  friendId: string;
  status: FriendshipStatus;
  isFavorite: boolean;
  isMuted: boolean;
  createdAt: Date;
  updatedAt: Date;
}

export interface IFriendshipRequest {
  _id: string;
  userId:
    | {
        _id: string;
        firstName: string;
        lastName: string;
        username: string;
        avatar: string;
      }
    | string;
  friendId:
    | {
        _id: string;
        firstName: string;
        lastName: string;
        username: string;
        avatar: string;
      }
    | string;
  status: FriendshipStatus;
  createdAt: Date;
  updatedAt: Date;
}

export interface IFindFriends {
  search?: string;
  userId: string;
  skip: number;
  limit: number;
  isFavorite?: boolean;
  isMuted?: boolean;
}

export interface IFindFriendshipRequests {
  requestType: FriendshipRequestType;
  skip: number;
  limit: number;
}

export interface ISendFriendshipRequest {
  userId: string;
  friendId: string;
}

export interface IAcceptFriendshipRequest {
  userId: string;
  friendshipId: string;
}

export interface IDeclineFriendshipRequest {
  userId: string;
  friendshipId: string;
}

export interface IRemoveFriendship {
  userId: string;
  friendshipId: string;
}

export interface ICancelFriendshipRequest {
  userId: string;
  friendshipId: string;
}

export interface IUpdateCloseFriend {
  userId: string;
  friendshipId: string;
  isCloseFriend: boolean;
}

export interface IUpdateNotification {
  userId: string;
  friendshipId: string;
  notification: boolean;
}

```

## File: connectfy-client/src/modules/users/MyFriends/api/api.ts
```typescript
import { baseQuery } from "@/common/api/axiosBaseQuery";
import { FriendshipStatus, RESOURCE } from "@/common/enums/enums";
import { createApi } from "@reduxjs/toolkit/query/react";
import { API_ENDPOINTS } from "@/common/constants/apiEndpoints";
import {
  IAcceptFriendshipRequest,
  ICancelFriendshipRequest,
  IDeclineFriendshipRequest,
  IFindFriends,
  IFindFriendshipRequests,
  IFriendship,
  IFriendshipRequest,
  IRemoveFriendship,
  IReturnedFriendship,
  ISendFriendshipRequest,
  IUpdateCloseFriend,
  IUpdateNotification,
} from "../types/types";
import { IFindAllResponse } from "@/common/interfaces/interfaces";
import { allUsersApi } from "../../AllUsers/api/api";
import { patchFriendshipNotificationMetadata } from "@/modules/notifications/api/api";
import { userProfileApi } from "../../UserProfile/api/api";
import { RootState } from "@/store/store";

/**
 * SENIOR HELPER: Bütün axtarış və siyahı keşlərini manual olaraq yeniləmək üçün utilit.
 * Bu funksiya verilmiş API-da bütün aktiv query-ləri gəzir və hədəf endpoint-ləri yeniləyir.
 */
const updateAllQueries = (
  state: any,
  dispatch: any,
  api: any,
  endpointName: string,
  updateFn: (draft: any, args: any) => void,
) => {
  const queries = state[api.reducerPath]?.queries || {};
  Object.values(queries).forEach((query: any) => {
    if (query?.endpointName === endpointName && query?.data) {
      dispatch(
        api.util.updateQueryData(
          endpointName,
          query.originalArgs,
          (draft: any) => updateFn(draft, query.originalArgs),
        ),
      );
    }
  });
};

export const myFriendsApi = createApi({
  reducerPath: RESOURCE.MY_FRIENDS,
  baseQuery: baseQuery,
  tagTypes: ["MyFriends", "FriendshipRequests", "UserProfile", "RequestCount"],
  endpoints: (builder) => ({
    // <==================== RELATIONSHIP ====================>

    // ======================== Find Friends
    findFriends: builder.query<IFindAllResponse<IFriendship>, IFindFriends>({
      query: (body) => ({
        url: API_ENDPOINTS.RELATIONSHIP.FRIENDSHIP.FIND_FRIENDS,
        method: "POST",
        body,
      }),
      providesTags: ["MyFriends"],
    }),

    // ======================== Find Friend Requests
    findFriendRequests: builder.query<
      IFindAllResponse<IFriendshipRequest>,
      IFindFriendshipRequests
    >({
      query: (body) => ({
        url: API_ENDPOINTS.RELATIONSHIP.FRIENDSHIP.FIND_REQUESTS,
        method: "POST",
        body,
      }),
      providesTags: ["FriendshipRequests"],
    }),

    // ======================== Find Online Friends
    findOnlineFriends: builder.query<any, { skip: number; limit: number; userId: string }>({
      query: (body) => ({
        url: API_ENDPOINTS.RELATIONSHIP.FRIENDSHIP.FIND_ONLINE_FRIENDS,
        method: "POST",
        body,
      }),
      providesTags: ["MyFriends"],
    }),

    // ======================== Find Suggestions
    findSuggestions: builder.query<any, { skip: number; limit: number; userId: string }>({
      query: (body) => ({
        url: API_ENDPOINTS.RELATIONSHIP.FRIENDSHIP.FIND_SUGGESTIONS,
        method: "POST",
        body,
      }),
      providesTags: ["MyFriends"],
    }),

    // ======================== Find Mutual Friends
    findMutualFriends: builder.query<any, { skip: number; limit: number; targetUserId: string }>({
      query: (body) => ({
        url: API_ENDPOINTS.RELATIONSHIP.FRIENDSHIP.FIND_MUTUAL_FRIENDS,
        method: "POST",
        body,
      }),
      providesTags: ["MyFriends"],
    }),

    // ======================== Send Friendship Request
    sendFriendRequest: builder.mutation<
      IReturnedFriendship,
      ISendFriendshipRequest
    >({
      query: (body) => ({
        url: API_ENDPOINTS.RELATIONSHIP.FRIENDSHIP.SEND_REQUEST,
        method: "POST",
        body,
      }),
      async onQueryStarted(
        { friendId },
        { dispatch, queryFulfilled, getState },
      ) {
        try {
          const { data: newFriendship } = await queryFulfilled;
          const state = getState() as RootState;

          dispatch(
            userProfileApi.util.updateQueryData(
              "findUser",
              friendId,
              (draft) => {
                if (draft) draft.relationship.friendship = newFriendship;
              },
            ),
          );
          updateAllQueries(state, dispatch, allUsersApi, "search", (draft) => {
            const user = draft.data.find((u: any) => u._id === friendId);
            if (user) user.relationship = newFriendship;
          });
        } catch (err: any) {
          // Sync Error Control
          const syncRel = err?.error?.additional?.relationship;
          if (syncRel) {
            const state = getState() as RootState;
            dispatch(
              userProfileApi.util.updateQueryData(
                "findUser",
                friendId,
                (draft) => {
                  if (draft) draft.relationship.friendship = syncRel;
                },
              ),
            );
            updateAllQueries(
              state,
              dispatch,
              allUsersApi,
              "search",
              (draft) => {
                const user = draft.data.find((u: any) => u._id === friendId);
                if (user) user.relationship = syncRel;
              },
            );
          }
        }
      },
    }),

    // ======================== Accept Friendship Request
    acceptFriendRequest: builder.mutation<
      IReturnedFriendship,
      IAcceptFriendshipRequest
    >({
      query: (body) => ({
        url: API_ENDPOINTS.RELATIONSHIP.FRIENDSHIP.ACCEPT_REQUEST,
        method: "POST",
        body,
      }),
      async onQueryStarted({ userId }, { dispatch, queryFulfilled, getState }) {
        try {
          const { data: newFriendship } = await queryFulfilled;
          const state = getState() as RootState;

          dispatch(
            userProfileApi.util.updateQueryData("findUser", userId, (draft) => {
              if (draft) draft.relationship.friendship = newFriendship;
            }),
          );
          updateAllQueries(state, dispatch, allUsersApi, "search", (draft) => {
            const user = draft.data.find((u: any) => u._id === userId);
            if (user) user.relationship = newFriendship;
          });
          updateAllQueries(
            state,
            dispatch,
            myFriendsApi,
            "findFriends",
            (draft) => {
              const exists = draft.data?.some(
                (f: any) => f._id === newFriendship._id,
              );
              if (!exists && draft.data) draft.data.push(newFriendship);
            },
          );
          updateAllQueries(
            state,
            dispatch,
            myFriendsApi,
            "findFriendRequests",
            (draft) => {
              if (draft.data) {
                draft.data = draft.data.filter(
                  (req: any) =>
                    req.userId?._id !== userId && req.friendId?._id !== userId,
                );
              }
            },
          );
          patchFriendshipNotificationMetadata(dispatch, state, userId, {
            isAccepted: true,
          });
        } catch (err: any) {
          const syncRel = err?.error?.additional?.relationship;
          const isCancelled = err?.error?.additional?.alreadyCancelled;
          if (syncRel) {
            const state = getState() as RootState;
            dispatch(
              userProfileApi.util.updateQueryData(
                "findUser",
                userId,
                (draft) => {
                  if (draft) draft.relationship.friendship = syncRel;
                },
              ),
            );
            updateAllQueries(
              state,
              dispatch,
              allUsersApi,
              "search",
              (draft) => {
                const user = draft.data.find((u: any) => u._id === userId);
                if (user) user.relationship = syncRel;
              },
            );
            patchFriendshipNotificationMetadata(dispatch, state, userId, {
              isAccepted:
                syncRel?.status === FriendshipStatus.Accepted || false,
              isDeclined:
                syncRel?.status === FriendshipStatus.Accepted ? false : true,
            });
          }
          if (isCancelled) {
            const state = getState() as RootState;
            dispatch(
              userProfileApi.util.updateQueryData(
                "findUser",
                userId,
                (draft) => {
                  if (draft) draft.relationship.friendship = null;
                },
              ),
            );
            updateAllQueries(
              state,
              dispatch,
              allUsersApi,
              "search",
              (draft) => {
                const user = draft.data.find((u: any) => u._id === userId);
                if (user) user.relationship = null;
              },
            );
            updateAllQueries(
              state,
              dispatch,
              myFriendsApi,
              "findFriends",
              (draft) => {
                if (draft.data)
                  draft.data = draft.data.filter(
                    (f: any) =>
                      f.userId !== userId && f.friendId?._id !== userId,
                  );
              },
            );
            patchFriendshipNotificationMetadata(dispatch, state, userId, {
              isDeclined: true,
            });
          }
        }
      },
    }),

    // ======================== Decline Friendship Request
    declineFriendRequest: builder.mutation<boolean, IDeclineFriendshipRequest>({
      query: (body) => ({
        url: API_ENDPOINTS.RELATIONSHIP.FRIENDSHIP.DECLINE_REQUEST,
        method: "POST",
        body,
      }),
      async onQueryStarted({ userId }, { dispatch, queryFulfilled, getState }) {
        try {
          await queryFulfilled;
          const state = getState() as RootState;

          dispatch(
            userProfileApi.util.updateQueryData("findUser", userId, (draft) => {
              if (draft) draft.relationship.friendship = null;
            }),
          );
          updateAllQueries(state, dispatch, allUsersApi, "search", (draft) => {
            const user = draft.data.find((u: any) => u._id === userId);
            if (user) user.relationship = null;
          });
          updateAllQueries(
            state,
            dispatch,
            myFriendsApi,
            "findFriendRequests",
            (draft) => {
              if (draft.data) {
                draft.data = draft.data.filter(
                  (req: any) =>
                    req.userId?._id !== userId && req.friendId?._id !== userId,
                );
              }
            },
          );
          patchFriendshipNotificationMetadata(dispatch, state, userId, {
            isDeclined: true,
          });
        } catch (err: any) {
          const additional = err?.error?.additional;
          const isCancelled = additional?.alreadyCancelled;
          const syncRel = additional?.relationship;
          if (isCancelled || syncRel) {
            const state = getState() as RootState;
            dispatch(
              userProfileApi.util.updateQueryData(
                "findUser",
                userId,
                (draft) => {
                  if (draft) draft.relationship.friendship = null;
                },
              ),
            );
            updateAllQueries(
              state,
              dispatch,
              allUsersApi,
              "search",
              (draft) => {
                const user = draft.data.find((u: any) => u._id === userId);
                if (user) user.relationship = null;
              },
            );
            updateAllQueries(
              state,
              dispatch,
              myFriendsApi,
              "findFriendRequests",
              (draft) => {
                if (draft.data)
                  draft.data = draft.data.filter(
                    (req: any) =>
                      req.userId?._id !== userId &&
                      req.friendId?._id !== userId,
                  );
              },
            );
            patchFriendshipNotificationMetadata(dispatch, state, userId, {
              isAccepted:
                syncRel?.status === FriendshipStatus.Accepted || false,
              isDeclined:
                syncRel?.status === FriendshipStatus.Accepted ? false : true,
            });
          }
        }
      },
    }),

    // ======================== Cancel Friendship Request
    cancelFriendRequest: builder.mutation<boolean, ICancelFriendshipRequest>({
      query: (body) => ({
        url: API_ENDPOINTS.RELATIONSHIP.FRIENDSHIP.CANCEL_REQUEST,
        method: "POST",
        body,
      }),
      async onQueryStarted({ userId }, { dispatch, queryFulfilled, getState }) {
        try {
          await queryFulfilled;
          const state = getState() as RootState;

          dispatch(
            userProfileApi.util.updateQueryData("findUser", userId, (draft) => {
              if (draft) draft.relationship.friendship = null;
            }),
          );
          updateAllQueries(state, dispatch, allUsersApi, "search", (draft) => {
            const user = draft.data.find((u: any) => u._id === userId);
            if (user) user.relationship = null;
          });
          updateAllQueries(
            state,
            dispatch,
            myFriendsApi,
            "findFriendRequests",
            (draft) => {
              if (draft.data)
                draft.data = draft.data.filter(
                  (req: any) =>
                    req.userId?._id !== userId && req.friendId?._id !== userId,
                );
            },
          );
        } catch (err: any) {
          const additional = err?.error?.additional;
          if (additional?.alreadyCancelled || additional?.relationship) {
            const state = getState() as RootState;
            dispatch(
              userProfileApi.util.updateQueryData(
                "findUser",
                userId,
                (draft) => {
                  if (draft)
                    draft.relationship.friendship =
                      additional.relationship || null;
                },
              ),
            );
            updateAllQueries(
              state,
              dispatch,
              allUsersApi,
              "search",
              (draft) => {
                const user = draft.data.find((u: any) => u._id === userId);
                if (user) user.relationship = additional.relationship || null;
              },
            );
            updateAllQueries(
              state,
              dispatch,
              myFriendsApi,
              "findFriendRequests",
              (draft) => {
                if (draft.data)
                  draft.data = draft.data.filter(
                    (req: any) =>
                      req.userId?._id !== userId &&
                      req.friendId?._id !== userId,
                  );
              },
            );
          }
        }
      },
    }),

    // ======================== Remove Friendship
    removeFriendship: builder.mutation<boolean, IRemoveFriendship>({
      query: (body) => ({
        url: API_ENDPOINTS.RELATIONSHIP.FRIENDSHIP.UNFRIEND,
        method: "POST",
        body,
      }),
      async onQueryStarted({ userId }, { dispatch, queryFulfilled, getState }) {
        try {
          await queryFulfilled;
          const state = getState() as RootState;

          dispatch(
            userProfileApi.util.updateQueryData("findUser", userId, (draft) => {
              if (draft) draft.relationship.friendship = null;
            }),
          );
          updateAllQueries(state, dispatch, allUsersApi, "search", (draft) => {
            const user = draft.data.find((u: any) => u._id === userId);
            if (user) user.relationship = null;
          });
          updateAllQueries(
            state,
            dispatch,
            myFriendsApi,
            "findFriends",
            (draft) => {
              if (draft.data)
                draft.data = draft.data.filter(
                  (f: any) => f.userId !== userId && f.friendId?._id !== userId,
                );
            },
          );
        } catch (err: any) {
          const isCancelled = err?.error?.additional?.alreadyCancelled;
          if (isCancelled) {
            const state = getState() as RootState;
            dispatch(
              userProfileApi.util.updateQueryData(
                "findUser",
                userId,
                (draft) => {
                  if (draft) draft.relationship.friendship = null;
                },
              ),
            );
            updateAllQueries(
              state,
              dispatch,
              allUsersApi,
              "search",
              (draft) => {
                const user = draft.data.find((u: any) => u._id === userId);
                if (user) user.relationship = null;
              },
            );
            updateAllQueries(
              state,
              dispatch,
              myFriendsApi,
              "findFriends",
              (draft) => {
                if (draft.data)
                  draft.data = draft.data.filter(
                    (f: any) =>
                      f.userId !== userId && f.friendId?._id !== userId,
                  );
              },
            );
          }
        }
      },
    }),

    // ======================== Update Close Friend
    updateCloseFriend: builder.mutation<boolean, IUpdateCloseFriend>({
      query: (body) => ({
        url: API_ENDPOINTS.RELATIONSHIP.FRIENDSHIP.UPDATE_CLOSE_FRIEND,
        method: "POST",
        body,
      }),
      async onQueryStarted(
        { userId, friendshipId, isCloseFriend },
        { dispatch, queryFulfilled, getState },
      ) {
        try {
          await queryFulfilled;
          const state = getState() as RootState;

          dispatch(
            userProfileApi.util.updateQueryData("findUser", userId, (draft) => {
              if (draft && draft.relationship.friendship)
                draft.relationship.friendship.isFavorite = isCloseFriend;
            }),
          );

          updateAllQueries(
            state,
            dispatch,
            myFriendsApi,
            "findFriends",
            (draft) => {
              const friendship = draft.find((f: any) => f._id === friendshipId);
              if (friendship) friendship.isCloseFriend = isCloseFriend;
            },
          );
        } catch (e) {
          console.error(e);
        }
      },
    }),

    // ======================== Update Notification
    updateNotification: builder.mutation<boolean, IUpdateNotification>({
      query: (body) => ({
        url: API_ENDPOINTS.RELATIONSHIP.FRIENDSHIP.UPDATE_NOTIFICATION,
        method: "POST",
        body,
      }),
      async onQueryStarted(
        { userId, friendshipId, notification },
        { dispatch, queryFulfilled, getState },
      ) {
        try {
          await queryFulfilled;
          const state = getState() as RootState;

          dispatch(
            userProfileApi.util.updateQueryData("findUser", userId, (draft) => {
              if (draft && draft.relationship.friendship)
                draft.relationship.friendship.isMuted = !notification;
            }),
          );

          updateAllQueries(
            state,
            dispatch,
            myFriendsApi,
            "findFriends",
            (draft) => {
              const friendship = draft.find((f: any) => f._id === friendshipId);
              if (friendship) friendship.isMuted = !notification;
            },
          );
        } catch (e) {
          console.error(e);
        }
      },
    }),

    // ======================== Requests Count
    requestsCount: builder.query<number, void>({
      query: () => ({
        url: API_ENDPOINTS.RELATIONSHIP.FRIENDSHIP.REQUESTS_COUNT,
        method: "POST",
      }),
      providesTags: ["RequestCount"],
    }),

    // <==================== RELATIONSHIP ====================>
  }),
});

export const {
  useFindFriendsQuery,
  useFindFriendRequestsQuery,
  useFindOnlineFriendsQuery,
  useFindSuggestionsQuery,
  useFindMutualFriendsQuery,

  useSendFriendRequestMutation,
  useAcceptFriendRequestMutation,
  useDeclineFriendRequestMutation,
  useCancelFriendRequestMutation,
  useRemoveFriendshipMutation,
  useUpdateCloseFriendMutation,
  useUpdateNotificationMutation,
  useRequestsCountQuery,
} = myFriendsApi;

```

## File: connectfy-client/src/modules/users/Blocklist/hooks/useBlocklist.tsx
```typescript
import { useErrors } from "@/hooks/useErrors";
import { useBlockUserMutation, useUnblockUserMutation } from "../api/api";
import { useTranslation } from "react-i18next";
import { snack } from "@/common/utils/snackManager";


export const useBlocklist = () => {
  const { showResponseErrors } = useErrors();
  const { t } = useTranslation();

  const [blockUser, { isLoading: isBlockUserLoading }] = useBlockUserMutation();
  const [unblockUser, { isLoading: isUnblockUserLoading }] = useUnblockUserMutation();

  const handleSyncError = (error: any) => {
    showResponseErrors(error);
  };

  const handleBlockUser = async (blockedId: string) => {
    try {
      await blockUser({ blockedId }).unwrap();
      snack.success(t("common.blocked_successfully", { defaultValue: "Blocked successfully" }));
    } catch (error) {
      handleSyncError(error);
    }
  };

  const handleUnblockUser = async (blockedId: string) => {
    try {
      await unblockUser({ blockedId }).unwrap();
      snack.success(t("common.unblocked_successfully", { defaultValue: "Unblocked successfully" }));
    } catch (error) {
      handleSyncError(error);
    }
  };

  return {
    handleBlockUser,
    handleUnblockUser,
    isActionLoading: isBlockUserLoading || isUnblockUserLoading,
    isBlockUserLoading,
    isUnblockUserLoading,
  };
};

```

## File: connectfy-client/src/modules/users/Blocklist/ui/BlockedUsers.tsx
```typescript
import { useUser } from "@/context/UserContext";
import { useFindBlockedUsersQuery } from "../api/api";
import { useTranslation } from "react-i18next";
import { useAppNavigation } from "@/hooks/useAppNavigation";
import { ROUTER } from "@/common/constants/routet";
import UniqueHeader from "@/components/Header/UnqiueHeader/UniqueHeader";
import Input from "@/components/ui/CustomInput/Input/Input";
import { Search } from "lucide-react";
import { useSearchParams } from "react-router-dom";
import { useCallback, useEffect, useState } from "react";
import Button from "@/components/ui/CustomButton/Button/Button";
import { checkEmptyString } from "@/common/utils/checkValues";
import { LoadingSpinner } from "@/components/Spinner/Settings/LoadingSpinner";
import UserCard from "@/components/Card/UserCard/UserCard";
import { motion } from "framer-motion";
import { useBlocklist } from "../hooks/useBlocklist";

const BlockedUsers = () => {
  const { user: currentUser } = useUser();
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();
  const { handleUnblockUser, isUnblockUserLoading } = useBlocklist();

  const [urlParams, setUrlParams] = useSearchParams();
  const queryParam = urlParams.get("q") || "";

  const [inputValue, setInputValue] = useState(queryParam);

  const { data, isLoading, isFetching } = useFindBlockedUsersQuery(
    {
      blockerId: currentUser?._id || "",
      skip: 1,
      limit: 10,
      search: queryParam,
    },
    {
      skip: !currentUser?._id,
      refetchOnMountOrArgChange: true,
    },
  );

  const executeSearch = useCallback(
    (value: string) => {
      const trimmed = value.trim();
      if (trimmed) {
        setUrlParams({ q: trimmed }, { replace: true });
      } else {
        setUrlParams({}, { replace: true });
      }
    },
    [setUrlParams],
  );

  const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    if (e.key === "Enter") executeSearch(inputValue);
  };

  useEffect(() => {
    setInputValue(queryParam);
  }, [queryParam]);

  const users = data?.data ?? [];

  return (
    <section className="w-full h-full pb-5 flex flex-col box-border bg-(--bg-color) text-(--text-color)">
      <UniqueHeader
        onClickBack={() => navigate(ROUTER.USERS.MAIN)}
        headerTitle={t("common.blocklist", { defaultValue: "Blocklist" })}
        headerSubtitle={t("common.blocklist_description", {
          defaultValue: "Manage your blocked users",
        })}
      />

      <div className="px-4 pt-1 pb-3">
        <div className="relative flex items-center gap-2">
          <Input
            id="user-search"
            type="search"
            inputSize="large"
            icon={<Search size={16} />}
            title={t("common.search")}
            value={inputValue}
            onChange={(e) => setInputValue(e.target.value)}
            onKeyDown={handleKeyDown}
            className="block w-full ps-9 bg-(--input-bg) border border-(--input-border) text-(--text-primary) text-sm rounded-xl focus:ring-(--primary-color) focus:border-(--primary-color) shadow-sm transition-colors"
          />
          <Button
            type="button"
            onClick={() => executeSearch(inputValue)}
            icon={<Search size={15} />}
            className="flex bg-(--btn-edit-bg) text-(--btn-edit-text) font-semibold p-5 rounded-xl items-center justify-center transition-all hover:scale-105 active:scale-95"
            disabled={!checkEmptyString(inputValue)}
          />
        </div>
      </div>

      <div className="flex-1 min-h-0 overflow-y-auto px-3">
        {isLoading || isFetching ? (
          <LoadingSpinner />
        ) : (
          <motion.ul
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            className="space-y-1"
          >
            {users.length > 0 ? (
              users.map((u: any, index: number) => (
                <motion.div
                  key={u._id}
                  initial={{ opacity: 0, y: 10 }}
                  animate={{ opacity: 1, y: 0 }}
                  transition={{ delay: index * 0.05 }}
                >
                  <UserCard
                    user={{
                      ...u.blocked,
                      avatar: {
                        key: null,
                        url: u.blocked?.avatar,
                        isCustom: false,
                      },
                    }}
                    actionButtons={
                      <Button
                        className="bg-(--btn-edit-bg) text-(--btn-edit-text) hover:bg-(--active-bg) px-4 py-2 rounded-lg text-sm font-medium transition-colors disabled:opacity-50 cursor-pointer"
                        title={t("common.unblock", { defaultValue: "Unblock" })}
                        onClick={() => handleUnblockUser(u.blocked?._id)}
                        disabled={isUnblockUserLoading}
                        hideTitleInMobile={false}
                      />
                    }
                  />
                </motion.div>
              ))
            ) : (
              <motion.div
                initial={{ opacity: 0, scale: 0.9 }}
                animate={{ opacity: 1, scale: 1 }}
                className="flex flex-col items-center justify-center h-48 gap-3 text-(--muted-color)"
              >
                <Search size={40} strokeWidth={1} className="opacity-30" />
                <p className="text-sm font-medium">
                  {queryParam
                    ? t("common.no_results_found", { query: queryParam })
                    : t("common.no_blocked_users", {
                        defaultValue: "No blocked users",
                      })}
                </p>
              </motion.div>
            )}
          </motion.ul>
        )}
      </div>
    </section>
  );
};

export default BlockedUsers;

```

## File: connectfy-client/src/modules/users/Blocklist/router/router.tsx
```typescript
import { RouteObject } from "react-router-dom";
import { lazy } from "react";
import { ROUTER } from "@/common/constants/routet";
import ComponentLoader from "@/components/Loader/Components/ComponentLoader";

const BlockedUsers = ComponentLoader(lazy(() => import("../ui/BlockedUsers")));

const routes: RouteObject[] = [
  {
    path: ROUTER.USERS.BLOCKLIST,
    element: <BlockedUsers />,
  },
];

export default routes;

```

## File: connectfy-client/src/modules/users/Blocklist/api/api.ts
```typescript
import { baseQuery } from "@/common/api/axiosBaseQuery";
import { RESOURCE } from "@/common/enums/enums";
import { createApi } from "@reduxjs/toolkit/query/react";
import { API_ENDPOINTS } from "@/common/constants/apiEndpoints";
import { IFindAllResponse } from "@/common/interfaces/interfaces";

import { userProfileApi } from "../../UserProfile/api/api";

export const blocklistApi = createApi({
  reducerPath: RESOURCE.BLOCKLIST,
  baseQuery: baseQuery,
  tagTypes: ["BlockedUsers"],
  endpoints: (builder) => ({
    findBlockedUsers: builder.query<IFindAllResponse<any>, any>({
      query: (body) => ({
        url: API_ENDPOINTS.RELATIONSHIP.BLOCKLIST.FIND_BLOCKED,
        method: "POST",
        body,
      }),
      providesTags: ["BlockedUsers"],
    }),

    unblockUser: builder.mutation<boolean, any>({
      query: (body) => ({
        url: API_ENDPOINTS.RELATIONSHIP.BLOCKLIST.UNBLOCK,
        method: "POST",
        body,
      }),
      async onQueryStarted(arg, { dispatch, queryFulfilled }) {
        try {
          await queryFulfilled;
          dispatch(
            userProfileApi.util.invalidateTags([
              { type: "UserProfile", id: arg.blockedId },
            ])
          );
        } catch {}
      },
      invalidatesTags: ["BlockedUsers"],
    }),

    blockUser: builder.mutation<boolean, any>({
      query: (body) => ({
        url: API_ENDPOINTS.RELATIONSHIP.BLOCKLIST.BLOCK,
        method: "POST",
        body,
      }),
      async onQueryStarted(arg, { dispatch, queryFulfilled }) {
        try {
          await queryFulfilled;
          dispatch(
            userProfileApi.util.invalidateTags([
              { type: "UserProfile", id: arg.blockedId },
            ])
          );
        } catch {}
      },
      invalidatesTags: ["BlockedUsers"],
    }),
  }),
});

export const { useFindBlockedUsersQuery, useUnblockUserMutation, useBlockUserMutation } = blocklistApi;

```

## File: connectfy-client/src/modules/users/FriendshipRequests/ui/FriendRequests.tsx
```typescript
import { useUser } from "@/context/UserContext";
import { useTranslation } from "react-i18next";
import { useAppNavigation } from "@/hooks/useAppNavigation";
import { ROUTER } from "@/common/constants/routet";
import UniqueHeader from "@/components/Header/UnqiueHeader/UniqueHeader";
import { Inbox, SendHorizontal, UserSearch, X } from "lucide-react";
import { useCallback, useState, useEffect } from "react";
import { LoadingSpinner } from "@/components/Spinner/Settings/LoadingSpinner";
import UserCard from "@/components/Card/UserCard/UserCard";
import { FriendshipRequestType, FriendshipStatus } from "@/common/enums/enums";
import FriendshipActions from "./components/FriendshipActions";
import { motion, AnimatePresence } from "framer-motion";
import { useIsMobile } from "@/hooks/useIsMobile";
import { useFriendship } from "../../MyFriends/hooks/useFriendship";
import { useFindFriendRequestsQuery } from "../../MyFriends/api/api";
import { ISearchUserResult } from "../../AllUsers/types/types";
import Button from "@/components/ui/CustomButton/Button/Button";
import NoProfilePhotoIcon from "@/assets/icons/NoProfilePhotoIcon";

const FriendRequets = () => {
  const { user: currentUser } = useUser();
  const { t } = useTranslation();
  const { navigate } = useAppNavigation();
  const isMobile = useIsMobile();

  const {
    handleAcceptFriendRequest,
    handleDeclineFriendRequest,
    handleCancelFriendRequest,
    isActionLoading,
  } = useFriendship();

  const [requestType, setRequestType] = useState<FriendshipRequestType>(
    FriendshipRequestType.Received,
  );

  const [lastAcceptedUser, setLastAcceptedUser] = useState<{
    fullName: string;
    avatar: string | null;
  } | null>(null);

  const { data, isLoading, isFetching } = useFindFriendRequestsQuery(
    { requestType: requestType, skip: 1, limit: 10 },
    { skip: !currentUser?._id, refetchOnMountOrArgChange: true },
  );

  useEffect(() => {
    if (lastAcceptedUser) {
      const timer = setTimeout(() => setLastAcceptedUser(null), 5000);
      return () => clearTimeout(timer);
    }
  }, [lastAcceptedUser]);

  const handleFilterToggle = useCallback((type: FriendshipRequestType) => {
    setRequestType(type);
  }, []);

  const users = data?.data ?? [];

  const baseFilterBtnClass =
    "relative flex items-center justify-center gap-2 px-4 py-2.5 text-sm font-semibold rounded-xl transition-colors duration-300 focus:outline-none select-none cursor-pointer w-full z-10";

  return (
    <section className="w-full h-full pb-5 flex flex-col box-border bg-(--bg-color) text-(--text-color)">
      <UniqueHeader
        onClickBack={() => navigate(ROUTER.USERS.MAIN)}
        headerTitle={t("common.friend_requests")}
        headerSubtitle={t("common.friend_requests_description")}
      />

      <div className="px-4 pt-2 pb-4">
        {/* Tablar */}
        <div className="flex justify-center w-full">
          <div className="relative flex w-full items-center p-1 bg-(--input-bg) border border-(--auth-glass-border) rounded-2xl shadow-sm gap-1">
            <motion.div
              layout
              transition={{ type: "spring", bounce: 0.2, duration: 0.6 }}
              className="absolute bg-(--primary-color) rounded-xl shadow-md"
              style={{
                width: "calc(50% - 6px)",
                height: "calc(100% - 8px)",
                left:
                  requestType === FriendshipRequestType.Received
                    ? "4px"
                    : "auto",
                right:
                  requestType === FriendshipRequestType.Sent ? "4px" : "auto",
              }}
            />
            <Button
              onClick={() => handleFilterToggle(FriendshipRequestType.Received)}
              className={`${baseFilterBtnClass} ${requestType === FriendshipRequestType.Received ? "text-white" : "text-(--muted-color)"}`}
            >
              <Inbox size={18} /> {!isMobile && t("common.received")}
            </Button>
            <Button
              onClick={() => handleFilterToggle(FriendshipRequestType.Sent)}
              className={`${baseFilterBtnClass} ${requestType === FriendshipRequestType.Sent ? "text-white" : "text-(--muted-color)"}`}
            >
              <SendHorizontal size={18} /> {!isMobile && t("common.sent")}
            </Button>
          </div>
        </div>
      </div>

      <AnimatePresence>
        {lastAcceptedUser && (
          <motion.div
            initial={{ height: 0, opacity: 0, marginBottom: 0 }}
            animate={{ height: "auto", opacity: 1, marginBottom: 16 }}
            exit={{ height: 0, opacity: 0, marginBottom: 0 }}
            className="overflow-hidden"
          >
            <div className="flex items-center justify-between p-4 bg-(--icon-green-bg) border border-(--icon-green-text) border-opacity-20 rounded-2xl shadow-sm">
              <div className="flex items-center gap-3">
                {lastAcceptedUser.avatar ? (
                  <img
                    src={lastAcceptedUser.avatar}
                    alt={lastAcceptedUser.fullName}
                    className="w-10 h-10 rounded-full object-cover"
                    loading="eager"
                    fetchPriority="high"
                    decoding="async"
                  />
                ) : (
                  <div className="flex items-center justify-center w-full h-full bg-(--active-bg-2)">
                    <NoProfilePhotoIcon />
                  </div>
                )}
                <p className="text-sm font-medium text-(--icon-green-text)">
                  {t("common.now_friends_with", {
                    name: lastAcceptedUser.fullName,
                  })}
                </p>
              </div>
              <Button
                onClick={() => setLastAcceptedUser(null)}
                className="text-(--icon-green-text) opacity-60 hover:opacity-100 transition-opacity"
                icon={<X size={18} />}
              />
            </div>
          </motion.div>
        )}
      </AnimatePresence>

      <div className="flex-1 min-h-0 overflow-y-auto px-3">
        {isLoading || isFetching ? (
          <div className="flex justify-center pt-10">
            <LoadingSpinner />
          </div>
        ) : (
          <AnimatePresence mode="wait">
            {users.length > 0 ? (
              <motion.ul
                key={requestType}
                initial={{ opacity: 0 }}
                animate={{ opacity: 1 }}
                className="space-y-2"
              >
                {users.map((u, index) => {
                  const isSentRequest = u.userId === currentUser?._id;
                  const targetUser = (
                    typeof u.friendId === "object" ? u.friendId : u.userId
                  ) as any;
                  if (!targetUser) return null;

                  const userData = {
                    ...targetUser,
                    _id:
                      targetUser._id ||
                      (typeof u.friendId === "string" ? u.friendId : u.userId),
                    avatar: {
                      key: null,
                      url: targetUser?.avatar || "",
                      isCustom: false,
                    },
                    relationship: {
                      status: FriendshipStatus.Accepted,
                      userId: u.userId,
                      friendId: targetUser._id,
                      _id: u._id,
                    },
                    friendshipRequest: false,
                  };

                  return (
                    <motion.div
                      key={u._id}
                      initial={{ opacity: 0, y: 15 }}
                      animate={{ opacity: 1, y: 0 }}
                      transition={{ delay: index * 0.05 }}
                    >
                      <UserCard
                        user={userData as ISearchUserResult}
                        actionButtons={
                          <FriendshipActions
                            relationship={{
                              ...userData.relationship,
                              status: FriendshipStatus.Pending,
                              createdAt: u.createdAt,
                              updatedAt: u.updatedAt,
                            }}
                            isSentRequest={isSentRequest}
                            onAccept={async (data) => {
                              await handleAcceptFriendRequest(data);
                              setLastAcceptedUser({
                                fullName: targetUser.fullName || `${targetUser.firstName} ${targetUser.lastName}`,
                                avatar: targetUser.avatar || null,
                              });
                            }}
                            onDecline={handleDeclineFriendRequest}
                            onCancel={handleCancelFriendRequest}
                            loading={isActionLoading}
                          />
                        }
                      />
                    </motion.div>
                  );
                })}
              </motion.ul>
            ) : (
              // Empty State (Dəyişməyib)
              <motion.div
                initial={{ opacity: 0 }}
                animate={{ opacity: 1 }}
                className="flex flex-col items-center justify-center h-64 gap-4 text-center px-10"
              >
                <div className="p-4 rounded-full bg-(--active-bg-2)">
                  <UserSearch
                    size={48}
                    className="text-(--primary-color) opacity-60"
                  />
                </div>
                <div>
                  <h3 className="text-base font-semibold text-(--text-primary)">
                    {requestType === FriendshipRequestType.Received
                      ? t("common.no_received_requests")
                      : t("common.no_sent_requests")}
                  </h3>
                  <p className="text-sm text-(--muted-color) mt-1">
                    {t("common.friend_requests_empty_hint")}
                  </p>
                </div>
                <Button
                  onClick={() => navigate(ROUTER.USERS.SEARCH)}
                  className="mt-2 px-6 py-2 bg-(--primary-color) text-white rounded-xl text-sm font-medium"
                  title={t("common.start_searching")}
                  hideTitleInMobile={false}
                />
              </motion.div>
            )}
          </AnimatePresence>
        )}
      </div>
    </section>
  );
};

export default FriendRequets;

```

## File: connectfy-client/src/modules/users/FriendshipRequests/ui/components/FriendshipActions.tsx
```typescript
import Button from "@/components/ui/CustomButton/Button/Button";
import { Check, Clock, X } from "lucide-react";
import { FriendshipStatus } from "@/common/enums/enums";
import { FC, Fragment } from "react";
import {
  IAcceptFriendshipRequest,
  IDeclineFriendshipRequest,
  IFriendship,
  IRemoveFriendship,
} from "@/modules/users/MyFriends/types/types";

interface IProps {
  relationship: Omit<IFriendship, "isFavorite" | "isMuted">;
  onCancel: (data: IRemoveFriendship) => void;
  onAccept: (data: IAcceptFriendshipRequest) => void;
  onDecline: (data: IDeclineFriendshipRequest) => void;
  loading: boolean;
  isSentRequest: boolean;
}

const FriendshipActions: FC<IProps> = ({
  relationship,
  onCancel,
  onAccept,
  onDecline,
  loading,
  isSentRequest,
}) => {
  const { _id, userId } = relationship;

  if (relationship?.status !== FriendshipStatus.Pending) {
    return null;
  }

  return (
    <Fragment>
      {isSentRequest ? (
        <Button
          className="p-2 rounded-lg border border-(--active-bg) opacity-70"
          icon={<Clock size={15} />}
          disabled={loading}
          isLoading={loading}
          onClick={() =>
            onCancel({
              friendshipId: _id,
              userId: (userId as Record<string, any>)._id,
            })
          }
        />
      ) : (
        <div className="flex items-center gap-2">
          <Button
            onClick={() =>
              onAccept({
                friendshipId: _id,
                userId: (userId as Record<string, any>)._id,
              })
            }
            className="p-2 rounded-lg bg-green-500/10 text-green-500 border border-green-500/50"
            disabled={loading}
            icon={<Check size={15} />}
            isLoading={loading}
          />
          <Button
            onClick={() =>
              onDecline({
                friendshipId: _id,
                userId: (userId as Record<string, any>)._id,
              })
            }
            className="p-2 rounded-lg bg-red-500/10 text-red-500 border border-red-500/50"
            disabled={loading}
            icon={<X size={15} />}
            isLoading={loading}
          />
        </div>
      )}
    </Fragment>
  );
};

export default FriendshipActions;

```

## File: connectfy-client/src/modules/users/FriendshipRequests/router/router.tsx
```typescript
import { RouteObject } from "react-router-dom";
import { lazy } from "react";
import { ROUTER } from "@/common/constants/routet";
import ComponentLoader from "@/components/Loader/Components/ComponentLoader";

const FriendRequests = ComponentLoader(
  lazy(() => import("../ui/FriendRequests")),
);

const routes: RouteObject[] = [
  {
    path: ROUTER.USERS.REQUESTS,
    element: <FriendRequests />,
  },
];

export default routes;

```

## File: connectfy-client/src/assets/icons/KeyIcon.tsx
```typescript
import { FC } from "react";

interface Props {
  color?: string;
}

const KeyIcon: FC<Props> = ({ color = "var(--border-color)" }) => {
  return (
    <svg
      className="MuiSvgIcon-root MuiSvgIcon-fontSizeSmall css-1bnpsma-MuiSvgIcon-root"
      focusable="false"
      aria-hidden="true"
      viewBox="0 0 24 24"
      data-testid="KeyIcon"
      style={{ color: color }}
    >
      <path d="M21 10h-8.35C11.83 7.67 9.61 6 7 6c-3.31 0-6 2.69-6 6s2.69 6 6 6c2.61 0 4.83-1.67 5.65-4H13l2 2 2-2 2 2 4-4.04zM7 15c-1.65 0-3-1.35-3-3s1.35-3 3-3 3 1.35 3 3-1.35 3-3 3"></path>
    </svg>
  );
};

export default KeyIcon;

```

## File: connectfy-client/src/assets/icons/FacebookIcon.tsx
```typescript
import { CSSProperties, FC } from "react";

interface IProps {
  style?: CSSProperties;
  className?: string;
  color?: string;
}

const FacebookIcon: FC<IProps> = ({
  style,
  className,
  color = "var(--text-color)",
}) => {
  return (
    <svg
      role="img"
      viewBox="0 0 24 24"
      xmlns="http://www.w3.org/2000/svg"
      style={style}
      className={className}
    >
      <title>Facebook</title>
      <path
        fill={color}
        d="M9.101 23.691v-7.98H6.627v-3.667h2.474v-1.58c0-4.085 1.848-5.978 5.858-5.978.401 0 .955.042 1.468.103a8.68 8.68 0 0 1 1.141.195v3.325a8.623 8.623 0 0 0-.653-.036 26.805 26.805 0 0 0-.733-.009c-.707 0-1.259.096-1.675.309a1.686 1.686 0 0 0-.679.622c-.258.42-.374.995-.374 1.752v1.297h3.919l-.386 2.103-.287 1.564h-3.246v8.245C19.396 23.238 24 18.179 24 12.044c0-6.627-5.373-12-12-12s-12 5.373-12 12c0 5.628 3.874 10.35 9.101 11.647Z"
      />
    </svg>
  );
};

export default FacebookIcon;

```

## File: connectfy-client/src/assets/icons/ErrorIcon.tsx
```typescript
const ErrorIcon = () => {
  return (
    <>
      <svg
        width="16"
        height="16"
        viewBox="0 0 16 16"
        fill="none"
        xmlns="http://www.w3.org/2000/svg"
      >
        <path
          fillRule="evenodd"
          clipRule="evenodd"
          d="M0.833496 8.00016C0.833496 11.9582 4.04212 15.1668 8.00016 15.1668C11.9582 15.1668 15.1668 11.9582 15.1668 8.00016C15.1668 4.04212 11.9582 0.833496 8.00016 0.833496C4.04212 0.833496 0.833496 4.04212 0.833496 8.00016ZM7.78808 7.35167C7.95278 7.37381 8.18061 7.43308 8.37394 7.62641C8.56726 7.81974 8.62654 8.04757 8.64868 8.21226C8.66702 8.3487 8.66692 8.50735 8.66684 8.6406L8.66683 8.66685V11.3335C8.66683 11.7017 8.36835 12.0002 8.00016 12.0002C7.63197 12.0002 7.3335 11.7017 7.3335 11.3335V8.66685C6.96531 8.66685 6.66683 8.36838 6.66683 8.00019C6.66683 7.632 6.96531 7.33352 7.3335 7.33352L7.35975 7.33351C7.493 7.33342 7.65165 7.33332 7.78808 7.35167ZM7.99712 4.66683C7.63058 4.66683 7.33344 4.96531 7.33344 5.3335C7.33344 5.70169 7.63058 6.00016 7.99712 6.00016H8.00309C8.36963 6.00016 8.66677 5.70169 8.66677 5.3335C8.66677 4.96531 8.36963 4.66683 8.00309 4.66683H7.99712Z"
          fill="#DE2C41"
        />
      </svg>
    </>
  );
};

export default ErrorIcon;

```

## File: connectfy-client/src/assets/icons/PinterestIcon.tsx
```typescript
import { CSSProperties, FC } from "react";

interface IProps {
  style?: CSSProperties;
  className?: string;
  color?: string;
}

const PinterestIcon: FC<IProps> = ({
  style,
  className,
  color = "var(--text-color)",
}) => {
  return (
    <svg
      role="img"
      viewBox="0 0 24 24"
      xmlns="http://www.w3.org/2000/svg"
      style={style}
      className={className}
    >
      <title>Pinterest</title>
      <path
        fill={color}
        d="M12.017 0C5.396 0 .029 5.367.029 11.987c0 5.079 3.158 9.417 7.618 11.162-.105-.949-.199-2.403.041-3.439.219-.937 1.406-5.957 1.406-5.957s-.359-.72-.359-1.781c0-1.663.967-2.911 2.168-2.911 1.024 0 1.518.769 1.518 1.688 0 1.029-.653 2.567-.992 3.992-.285 1.193.6 2.165 1.775 2.165 2.128 0 3.768-2.245 3.768-5.487 0-2.861-2.063-4.869-5.008-4.869-3.41 0-5.409 2.562-5.409 5.199 0 1.033.394 2.143.889 2.741.099.12.112.225.085.345-.09.375-.293 1.199-.334 1.363-.053.225-.172.271-.401.165-1.495-.69-2.433-2.878-2.433-4.646 0-3.776 2.748-7.252 7.92-7.252 4.158 0 7.392 2.967 7.392 6.923 0 4.135-2.607 7.462-6.233 7.462-1.214 0-2.354-.629-2.758-1.379l-.749 2.848c-.269 1.045-1.004 2.352-1.498 3.146 1.123.345 2.306.535 3.55.535 6.607 0 11.985-5.365 11.985-11.987C23.97 5.39 18.592.026 11.985.026L12.017 0z"
      />
    </svg>
  );
};

export default PinterestIcon;

```

## File: connectfy-client/src/assets/icons/MainIcon.tsx
```typescript
import { CSSProperties } from "react";

interface MainIconInterface {
  styles?: CSSProperties;
  className?: string;
}

export default function MainIcon({
  styles,
  className = "",
}: MainIconInterface) {
  return (
    <svg
      viewBox="0 0 576 512"
      aria-hidden="true"
      role="img"
      style={{ ...styles, color: styles?.color ?? "var(--primary-color)" }}
      className={className}
    >
      <path
        fill="currentColor"
        d="M416 192c0-88.4-93.1-160-208-160S0 103.6 0 192c0 34.3 14.1 65.9 38 92-13.4 30.2-35.5 54.2-35.8 54.5-2.2 2.3-2.8 5.7-1.5 8.7S4.8 352 8 352c36.6 0 66.9-12.3 88.7-25 32.2 15.7 70.3 25 111.3 25 114.9 0 208-71.6 208-160zm122 220c23.9-26 38-57.7 38-92 0-66.9-53.5-124.2-129.3-148.1.9 6.6 1.3 13.3 1.3 20.1 0 105.9-107.7 192-240 192-10.8 0-21.3-.8-31.7-1.9C207.8 439.6 281.8 480 368 480c41 0 79.1-9.2 111.3-25 21.8 12.7 52.1 25 88.7 25 3.2 0 6.1-1.9 7.3-4.8 1.3-2.9.7-6.3-1.5-8.7-.3-.3-22.4-24.2-35.8-54.5z"
      />
    </svg>
  );
}

```

## File: connectfy-client/src/assets/icons/GoogleIcon.tsx
```typescript
const GoogleIcon = () => {
  return (
    <svg className="size-5" viewBox="0 0 24 24">
      <path
        d="M22.56 12.25c0-.78-.07-1.53-.2-2.25H12v4.26h5.92c-.26 1.37-1.04 2.53-2.21 3.31v2.77h3.57c2.08-1.92 3.28-4.74 3.28-8.09z"
        fill="#4285F4"
      ></path>
      <path
        d="M12 23c2.97 0 5.46-.98 7.28-2.66l-3.57-2.77c-.98.66-2.23 1.06-3.71 1.06-2.86 0-5.29-1.93-6.16-4.53H2.18v2.84C3.99 20.53 7.7 23 12 23z"
        fill="#34A853"
      ></path>
      <path
        d="M5.84 14.09c-.22-.66-.35-1.36-.35-2.09s.13-1.43.35-2.09V7.07H2.18C1.43 8.55 1 10.22 1 12s.43 3.45 1.18 4.93l2.85-2.22.81-.62z"
        fill="#FBBC05"
      ></path>
      <path
        d="M12 5.38c1.62 0 3.06.56 4.21 1.66l3.15-3.15C17.45 2.09 14.97 1 12 1 7.7 1 3.99 3.47 2.18 7.07l3.66 2.84c.87-2.6 3.3-4.53 6.16-4.53z"
        fill="#EA4335"
      ></path>
    </svg>
  );
};

export default GoogleIcon;

```

## File: connectfy-client/src/assets/icons/LinkedInIcon.tsx
```typescript
import { CSSProperties, FC } from "react";

interface IProps {
  style?: CSSProperties;
  className?: string;
  color?: string;
}

const LinkedInIcon: FC<IProps> = ({
  style,
  className,
  color = "var(--text-color)",
}) => {
  return (
    <svg
      role="img"
      viewBox="0 0 24 24"
      xmlns="http://www.w3.org/2000/svg"
      style={style}
      className={className}
    >
      <title>LinkedIn</title>
      <path
        fill={color}
        d="M20.447 20.452h-3.554v-5.569c0-1.328-.027-3.037-1.852-3.037-1.853 0-2.136 1.445-2.136 2.939v5.667H9.351V9h3.414v1.561h.046c.477-.9 1.637-1.85 3.37-1.85 3.601 0 4.267 2.37 4.267 5.455v6.286zM5.337 7.433c-1.144 0-2.063-.926-2.063-2.065 0-1.138.92-2.063 2.063-2.063 1.14 0 2.064.925 2.064 2.063 0 1.139-.925 2.065-2.064 2.065zm1.782 13.019H3.555V9h3.564v11.452zM22.225 0H1.771C.792 0 0 .774 0 1.729v20.542C0 23.227.792 24 1.771 24h20.451C23.2 24 24 23.227 24 22.271V1.729C24 .774 23.2 0 22.222 0h.003z"
      />
    </svg>
  );
};

export default LinkedInIcon;

```

## File: connectfy-client/src/assets/icons/DiscordIcon.tsx
```typescript
import { CSSProperties, FC } from "react";

interface IProps {
  style?: CSSProperties;
  className?: string;
  color?: string;
}

const DiscordIcon: FC<IProps> = ({
  style,
  className,
  color = "var(--text-color)",
}) => {
  return (
    <svg
      role="img"
      viewBox="0 0 24 24"
      xmlns="http://www.w3.org/2000/svg"
      style={style}
      className={className}
    >
      <title>Discord</title>
      <path
        fill={color}
        d="M20.317 4.3698a19.7913 19.7913 0 00-4.8851-1.5152.0741.0741 0 00-.0785.0371c-.211.3753-.4447.8648-.6083 1.2495-1.8447-.2762-3.68-.2762-5.4868 0-.1636-.3933-.4058-.8742-.6177-1.2495a.077.077 0 00-.0785-.037 19.7363 19.7363 0 00-4.8852 1.515.0699.0699 0 00-.0321.0277C.5334 9.0458-.319 13.5799.0992 18.0578a.0824.0824 0 00.0312.0561c2.0528 1.5076 4.0413 2.4228 5.9929 3.0294a.0777.0777 0 00.0842-.0276c.4616-.6304.8731-1.2952 1.226-1.9942a.076.076 0 00-.0416-.1057c-.6528-.2476-1.2743-.5495-1.8722-.8923a.077.077 0 01-.0076-.1277c.1258-.0943.2517-.1923.3718-.2914a.0743.0743 0 01.0776-.0105c3.9278 1.7933 8.18 1.7933 12.0614 0a.0739.0739 0 01.0785.0095c.1202.099.246.1981.3728.2924a.077.077 0 01-.0066.1276 12.2986 12.2986 0 01-1.873.8914.0766.0766 0 00-.0407.1067c.3604.698.7719 1.3628 1.225 1.9932a.076.076 0 00.0842.0286c1.961-.6067 3.9495-1.5219 6.0023-3.0294a.077.077 0 00.0313-.0552c.5004-5.177-.8382-9.6739-3.5485-13.6604a.061.061 0 00-.0312-.0286zM8.02 15.3312c-1.1825 0-2.1569-1.0857-2.1569-2.419 0-1.3332.9555-2.4189 2.157-2.4189 1.2108 0 2.1757 1.0952 2.1568 2.419 0 1.3332-.9555 2.4189-2.1569 2.4189zm7.9748 0c-1.1825 0-2.1569-1.0857-2.1569-2.419 0-1.3332.9554-2.4189 2.1569-2.4189 1.2108 0 2.1757 1.0952 2.1568 2.419 0 1.3332-.946 2.4189-2.1568 2.4189Z"
      />
    </svg>
  );
};

export default DiscordIcon;

```

## File: connectfy-client/src/assets/icons/RedditIcon.tsx
```typescript
import { CSSProperties, FC } from "react";

interface IProps {
  style?: CSSProperties;
  className?: string;
  color?: string;
}

const RedditIcon: FC<IProps> = ({
  style,
  className,
  color = "var(--text-color)",
}) => {
  return (
    <svg
      role="img"
      viewBox="0 0 24 24"
      xmlns="http://www.w3.org/2000/svg"
      style={style}
      className={className}
    >
      <title>Reddit</title>
      <path
        fill={color}
        d="M12 0C5.373 0 0 5.373 0 12c0 3.314 1.343 6.314 3.515 8.485l-2.286 2.286C.775 23.225 1.097 24 1.738 24H12c6.627 0 12-5.373 12-12S18.627 0 12 0Zm4.388 3.199c1.104 0 1.999.895 1.999 1.999 0 1.105-.895 2-1.999 2-.946 0-1.739-.657-1.947-1.539v.002c-1.147.162-2.032 1.15-2.032 2.341v.007c1.776.067 3.4.567 4.686 1.363.473-.363 1.064-.58 1.707-.58 1.547 0 2.802 1.254 2.802 2.802 0 1.117-.655 2.081-1.601 2.531-.088 3.256-3.637 5.876-7.997 5.876-4.361 0-7.905-2.617-7.998-5.87-.954-.447-1.614-1.415-1.614-2.538 0-1.548 1.255-2.802 2.803-2.802.645 0 1.239.218 1.712.585 1.275-.79 2.881-1.291 4.64-1.365v-.01c0-1.663 1.263-3.034 2.88-3.207.188-.911.993-1.595 1.959-1.595Zm-8.085 8.376c-.784 0-1.459.78-1.506 1.797-.047 1.016.64 1.429 1.426 1.429.786 0 1.371-.369 1.418-1.385.047-1.017-.553-1.841-1.338-1.841Zm7.406 0c-.786 0-1.385.824-1.338 1.841.047 1.017.634 1.385 1.418 1.385.785 0 1.473-.413 1.426-1.429-.046-1.017-.721-1.797-1.506-1.797Zm-3.703 4.013c-.974 0-1.907.048-2.77.135-.147.015-.241.168-.183.305.483 1.154 1.622 1.964 2.953 1.964 1.33 0 2.47-.81 2.953-1.964.057-.137-.037-.29-.184-.305-.863-.087-1.795-.135-2.769-.135Z"
      />
    </svg>
  );
};

export default RedditIcon;

```

## File: connectfy-client/src/assets/icons/SnapchatIcon.tsx
```typescript
import { CSSProperties, FC } from "react";

interface IProps {
  style?: CSSProperties;
  className?: string;
  color?: string;
}

const SnapchatIcon: FC<IProps> = ({
  style,
  className,
  color = "var(--text-color)",
}) => {
  return (
    <svg
      role="img"
      viewBox="0 0 24 24"
      xmlns="http://www.w3.org/2000/svg"
      style={style}
      className={className}
    >
      <title>Snapchat</title>
      <path
        fill={color}
        d="M12.206.793c.99 0 4.347.276 5.93 3.821.529 1.193.403 3.219.299 4.847l-.003.06c-.012.18-.022.345-.03.51.075.045.203.09.401.09.3-.016.659-.12 1.033-.301.165-.088.344-.104.464-.104.182 0 .359.029.509.09.45.149.734.479.734.838.015.449-.39.839-1.213 1.168-.089.029-.209.075-.344.119-.45.135-1.139.36-1.333.81-.09.224-.061.524.12.868l.015.015c.06.136 1.526 3.475 4.791 4.014.255.044.435.27.42.509 0 .075-.015.149-.045.225-.24.569-1.273.988-3.146 1.271-.059.091-.12.375-.164.57-.029.179-.074.36-.134.553-.076.271-.27.405-.555.405h-.03c-.135 0-.313-.031-.538-.074-.36-.075-.765-.135-1.273-.135-.3 0-.599.015-.913.074-.6.104-1.123.464-1.723.884-.853.599-1.826 1.288-3.294 1.288-.06 0-.119-.015-.18-.015h-.149c-1.468 0-2.427-.675-3.279-1.288-.599-.42-1.107-.779-1.707-.884-.314-.045-.629-.074-.928-.074-.54 0-.958.089-1.272.149-.211.043-.391.074-.54.074-.374 0-.523-.224-.583-.42-.061-.192-.09-.389-.135-.567-.046-.181-.105-.494-.166-.57-1.918-.222-2.95-.642-3.189-1.226-.031-.063-.052-.15-.055-.225-.015-.243.165-.465.42-.509 3.264-.54 4.73-3.879 4.791-4.02l.016-.029c.18-.345.224-.645.119-.869-.195-.434-.884-.658-1.332-.809-.121-.029-.24-.074-.346-.119-1.107-.435-1.257-.93-1.197-1.273.09-.479.674-.793 1.168-.793.146 0 .27.029.383.074.42.194.789.3 1.104.3.234 0 .384-.06.465-.105l-.046-.569c-.098-1.626-.225-3.651.307-4.837C7.392 1.077 10.739.807 11.727.807l.419-.015h.06z"
      />
    </svg>
  );
};

export default SnapchatIcon;

```

## File: connectfy-client/src/assets/icons/XIcon.tsx
```typescript
import { CSSProperties, FC } from "react";

interface IProps {
  style?: CSSProperties;
  className?: string;
  color?: string;
}

const XIcon: FC<IProps> = ({
  style,
  className,
  color = "var(--text-color)",
}) => {
  return (
    <svg
      role="img"
      viewBox="0 0 24 24"
      xmlns="http://www.w3.org/2000/svg"
      style={style}
      className={className}
    >
      <title>X</title>
      <path
        fill={color}
        d="M14.234 10.162 22.977 0h-2.072l-7.591 8.824L7.251 0H.258l9.168 13.343L.258 24H2.33l8.016-9.318L16.749 24h6.993zm-2.837 3.299-.929-1.329L3.076 1.56h3.182l5.965 8.532.929 1.329 7.754 11.09h-3.182z"
      />
    </svg>
  );
};

export default XIcon;

```

## File: connectfy-client/src/assets/icons/LockIcon.tsx
```typescript
import { FC } from "react";

interface Props {
  color?: string;
}

const LockIcon: FC<Props> = ({ color }) => {
  return (
    <svg
      data-v-6433c584=""
      xmlns="http://www.w3.org/2000/svg"
      width="24"
      height="24"
      viewBox="0 0 24 24"
      fill="none"
      stroke="currentColor"
      stroke-width="2"
      stroke-linecap="round"
      stroke-linejoin="round"
      className="lucide lucide-lock-keyhole-icon lucide-lock-keyhole"
      aria-hidden="true"
      style={{ color: color }}
    >
      <circle cx="12" cy="16" r="1"></circle>
      <rect x="3" y="10" width="18" height="12" rx="2"></rect>
      <path d="M7 10V7a5 5 0 0 1 10 0v3"></path>
    </svg>
  );
};

export default LockIcon;

```

## File: connectfy-client/src/assets/icons/TiktokIcon.tsx
```typescript
import { CSSProperties, FC } from "react";

interface IProps {
  style?: CSSProperties;
  className?: string;
  color?: string;
}

const TiktokIcon: FC<IProps> = ({
  style,
  className,
  color = "var(--text-color)",
}) => {
  return (
    <svg
      role="img"
      viewBox="0 0 24 24"
      xmlns="http://www.w3.org/2000/svg"
      style={style}
      className={className}
    >
      <title>TikTok</title>
      <path
        fill={color}
        d="M12.525.02c1.31-.02 2.61-.01 3.91-.02.08 1.53.63 3.09 1.75 4.17 1.12 1.11 2.7 1.62 4.24 1.79v4.03c-1.44-.05-2.89-.35-4.2-.97-.57-.26-1.1-.59-1.62-.93-.01 2.92.01 5.84-.02 8.75-.08 1.4-.54 2.79-1.35 3.94-1.31 1.92-3.58 3.17-5.91 3.21-1.43.08-2.86-.31-4.08-1.03-2.02-1.19-3.44-3.37-3.65-5.71-.02-.5-.03-1-.01-1.49.18-1.9 1.12-3.72 2.58-4.96 1.66-1.44 3.98-2.13 6.15-1.72.02 1.48-.04 2.96-.04 4.44-.99-.32-2.15-.23-3.02.37-.63.41-1.11 1.04-1.36 1.75-.21.51-.15 1.07-.14 1.61.24 1.64 1.82 3.02 3.5 2.87 1.12-.01 2.19-.66 2.77-1.61.19-.33.4-.67.41-1.06.1-1.79.06-3.57.07-5.36.01-4.03-.01-8.05.02-12.07z"
      />
    </svg>
  );
};

export default TiktokIcon;

```

## File: connectfy-client/src/assets/icons/GithubIcon.tsx
```typescript
import { CSSProperties, FC } from "react";

interface IProps {
  style?: CSSProperties;
  className?: string;
  color?: string;
}

const GithubIcon: FC<IProps> = ({
  style,
  className,
  color = "var(--text-color)",
}) => {
  return (
    <svg
      role="img"
      viewBox="0 0 24 24"
      xmlns="http://www.w3.org/2000/svg"
      style={style}
      className={className}
    >
      <title>GitHub</title>
      <path
        fill={color}
        d="M12 .297c-6.63 0-12 5.373-12 12 0 5.303 3.438 9.8 8.205 11.385.6.113.82-.258.82-.577 0-.285-.01-1.04-.015-2.04-3.338.724-4.042-1.61-4.042-1.61C4.422 18.07 3.633 17.7 3.633 17.7c-1.087-.744.084-.729.084-.729 1.205.084 1.838 1.236 1.838 1.236 1.07 1.835 2.809 1.305 3.495.998.108-.776.417-1.305.76-1.605-2.665-.3-5.466-1.332-5.466-5.93 0-1.31.465-2.38 1.235-3.22-.135-.303-.54-1.523.105-3.176 0 0 1.005-.322 3.3 1.23.96-.267 1.98-.399 3-.405 1.02.006 2.04.138 3 .405 2.28-1.552 3.285-1.23 3.285-1.23.645 1.653.24 2.873.12 3.176.765.84 1.23 1.91 1.23 3.22 0 4.61-2.805 5.625-5.475 5.92.42.36.81 1.096.81 2.22 0 1.606-.015 2.896-.015 3.286 0 .315.21.69.825.57C20.565 22.092 24 17.592 24 12.297c0-6.627-5.373-12-12-12"
      />
    </svg>
  );
};

export default GithubIcon;

```

## File: connectfy-client/src/assets/icons/NoProfilePhotoIcon.tsx
```typescript
const NoProfilePhotoIcon = () => {
  return (
    <svg
      viewBox="0 0 24 24"
      fill="none"
      stroke="currentColor"
      strokeWidth="1.5"
      strokeLinecap="round"
      strokeLinejoin="round"
      className="w-1/2 h-1/2 text-(--muted-color) opacity-50"
    >
      <path d="M19 21v-2a4 4 0 0 0-4-4H9a4 4 0 0 0-4 4v2" />
      <circle cx="12" cy="7" r="4" />
    </svg>
  );
};

export default NoProfilePhotoIcon;

```

## File: connectfy-client/src/assets/icons/EyeIcon.tsx
```typescript
import { FC } from "react";

interface Props {
  color?: string;
}

const EyeIcon: FC<Props> = ({ color = "var(--border-color)" }) => {
  return (
    <svg
      data-v-6433c584=""
      xmlns="http://www.w3.org/2000/svg"
      width="24"
      height="24"
      viewBox="0 0 24 24"
      fill="none"
      stroke="currentColor"
      stroke-width="2"
      stroke-linecap="round"
      stroke-linejoin="round"
      className="lucide lucide-eye-icon lucide-eye"
      aria-hidden="true"
      style={{ color: color }}
    >
      <path d="M2.062 12.348a1 1 0 0 1 0-.696 10.75 10.75 0 0 1 19.876 0 1 1 0 0 1 0 .696 10.75 10.75 0 0 1-19.876 0"></path>
      <circle cx="12" cy="12" r="3"></circle>
    </svg>
  );
};

export default EyeIcon;

```

## File: connectfy-client/src/assets/icons/WhatsappIcon.tsx
```typescript
import { CSSProperties, FC } from "react";

interface IProps {
  style?: CSSProperties;
  className?: string;
  color?: string;
}

const WhatsappIcon: FC<IProps> = ({
  style,
  className,
  color = "var(--text-color)",
}) => {
  return (
    <svg
      role="img"
      viewBox="0 0 24 24"
      xmlns="http://www.w3.org/2000/svg"
      style={style}
      className={className}
    >
      <title>WhatsApp</title>
      <path
        fill={color}
        d="M17.472 14.382c-.297-.149-1.758-.867-2.03-.967-.273-.099-.471-.148-.67.15-.197.297-.767.966-.94 1.164-.173.199-.347.223-.644.075-.297-.15-1.255-.463-2.39-1.475-.883-.788-1.48-1.761-1.653-2.059-.173-.297-.018-.458.13-.606.134-.133.298-.347.446-.52.149-.174.198-.298.298-.497.099-.198.05-.371-.025-.52-.075-.149-.669-1.612-.916-2.207-.242-.579-.487-.5-.669-.51-.173-.008-.371-.01-.57-.01-.198 0-.52.074-.792.372-.272.297-1.04 1.016-1.04 2.479 0 1.462 1.065 2.875 1.213 3.074.149.198 2.096 3.2 5.077 4.487.709.306 1.262.489 1.694.625.712.227 1.36.195 1.871.118.571-.085 1.758-.719 2.006-1.413.248-.694.248-1.289.173-1.413-.074-.124-.272-.198-.57-.347m-5.421 7.403h-.004a9.87 9.87 0 01-5.031-1.378l-.361-.214-3.741.982.998-3.648-.235-.374a9.86 9.86 0 01-1.51-5.26c.001-5.45 4.436-9.884 9.888-9.884 2.64 0 5.122 1.03 6.988 2.898a9.825 9.825 0 012.893 6.994c-.003 5.45-4.437 9.884-9.885 9.884m8.413-18.297A11.815 11.815 0 0012.05 0C5.495 0 .16 5.335.157 11.892c0 2.096.547 4.142 1.588 5.945L.057 24l6.305-1.654a11.882 11.882 0 005.683 1.448h.005c6.554 0 11.89-5.335 11.893-11.893a11.821 11.821 0 00-3.48-8.413Z"
      />
    </svg>
  );
};

export default WhatsappIcon;

```

## File: connectfy-client/src/assets/icons/BehanceIcon.tsx
```typescript
import { CSSProperties, FC } from "react";

interface IProps {
  style?: CSSProperties;
  className?: string;
  color?: string;
}

const XIcon: FC<IProps> = ({
  style,
  className,
  color = "var(--text-color)",
}) => {
  return (
    <svg
      role="img"
      viewBox="0 0 24 24"
      xmlns="http://www.w3.org/2000/svg"
      style={style}
      className={className}
    >
      <title>Behance</title>
      <path
        fill={color}
        d="M16.969 16.927a2.561 2.561 0 0 0 1.901.677 2.501 2.501 0 0 0 1.531-.475c.362-.235.636-.584.779-.99h2.585a5.091 5.091 0 0 1-1.9 2.896 5.292 5.292 0 0 1-3.091.88 5.839 5.839 0 0 1-2.284-.433 4.871 4.871 0 0 1-1.723-1.211 5.657 5.657 0 0 1-1.08-1.874 7.057 7.057 0 0 1-.383-2.393c-.005-.8.129-1.595.396-2.349a5.313 5.313 0 0 1 5.088-3.604 4.87 4.87 0 0 1 2.376.563c.661.362 1.231.87 1.668 1.485a6.2 6.2 0 0 1 .943 2.133c.194.821.263 1.666.205 2.508h-7.699c-.063.79.184 1.574.688 2.187ZM6.947 4.084a8.065 8.065 0 0 1 1.928.198 4.29 4.29 0 0 1 1.49.638c.418.303.748.711.958 1.182.241.579.357 1.203.341 1.83a3.506 3.506 0 0 1-.506 1.961 3.726 3.726 0 0 1-1.503 1.287 3.588 3.588 0 0 1 2.027 1.437c.464.747.697 1.615.67 2.494a4.593 4.593 0 0 1-.423 2.032 3.945 3.945 0 0 1-1.163 1.413 5.114 5.114 0 0 1-1.683.807 7.135 7.135 0 0 1-1.928.259H0V4.084h6.947Zm-.235 12.9c.308.004.616-.029.916-.099a2.18 2.18 0 0 0 .766-.332c.228-.158.411-.371.534-.619.142-.317.208-.663.191-1.009a2.08 2.08 0 0 0-.642-1.715 2.618 2.618 0 0 0-1.696-.505h-3.54v4.279h3.471Zm13.635-5.967a2.13 2.13 0 0 0-1.654-.619 2.336 2.336 0 0 0-1.163.259 2.474 2.474 0 0 0-.738.62 2.359 2.359 0 0 0-.396.792c-.074.239-.12.485-.137.734h4.769a3.239 3.239 0 0 0-.679-1.785l-.002-.001Zm-13.813-.648a2.254 2.254 0 0 0 1.423-.433c.399-.355.607-.88.56-1.413a1.916 1.916 0 0 0-.178-.891 1.298 1.298 0 0 0-.495-.533 1.851 1.851 0 0 0-.711-.274 3.966 3.966 0 0 0-.835-.073H3.241v3.631h3.293v-.014ZM21.62 5.122h-5.976v1.527h5.976V5.122Z"
      />
    </svg>
  );
};

export default XIcon;

```

## File: connectfy-client/src/assets/icons/YoutubeIcon.tsx
```typescript
import { CSSProperties, FC } from "react";

interface IProps {
  style?: CSSProperties;
  className?: string;
  color?: string;
}

const YoutubeIcon: FC<IProps> = ({
  style,
  className,
  color = "var(--text-color)",
}) => {
  return (
    <svg
      role="img"
      viewBox="0 0 24 24"
      xmlns="http://www.w3.org/2000/svg"
      style={style}
      className={className}
    >
      <title>YouTube</title>
      <path
        fill={color}
        d="M23.498 6.186a3.016 3.016 0 0 0-2.122-2.136C19.505 3.545 12 3.545 12 3.545s-7.505 0-9.377.505A3.017 3.017 0 0 0 .502 6.186C0 8.07 0 12 0 12s0 3.93.502 5.814a3.016 3.016 0 0 0 2.122 2.136c1.871.505 9.376.505 9.376.505s7.505 0 9.377-.505a3.015 3.015 0 0 0 2.122-2.136C24 15.93 24 12 24 12s0-3.93-.502-5.814zM9.545 15.568V8.432L15.818 12l-6.273 3.568z"
      />
    </svg>
  );
};

export default YoutubeIcon;

```

## File: connectfy-client/src/assets/icons/EyeCloseIcon.tsx
```typescript
import { FC } from "react";

interface Props {
  color?: string;
}

const EyeCloseIcon: FC<Props> = ({ color = "var(--border-color)" }) => {
  return (
    <svg
      data-v-6433c584=""
      xmlns="http://www.w3.org/2000/svg"
      width="24"
      height="24"
      viewBox="0 0 24 24"
      fill="none"
      stroke="currentColor"
      stroke-width="2"
      stroke-linecap="round"
      stroke-linejoin="round"
      className="lucide lucide-eye-closed-icon lucide-eye-closed"
      aria-hidden="true"
      style={{ color: color }}
    >
      <path d="m15 18-.722-3.25"></path>
      <path d="M2 8a10.645 10.645 0 0 0 20 0"></path>
      <path d="m20 15-1.726-2.05"></path>
      <path d="m4 15 1.726-2.05"></path>
      <path d="m9 18 .722-3.25"></path>
    </svg>
  );
};

export default EyeCloseIcon;

```

## File: connectfy-client/src/assets/icons/MediumIcon.tsx
```typescript
import { CSSProperties, FC } from "react";

interface IProps {
  style?: CSSProperties;
  className?: string;
  color?: string;
}

const MediumIcon: FC<IProps> = ({
  style,
  className,
  color = "var(--text-color)",
}) => {
  return (
    <svg
      role="img"
      viewBox="0 0 24 24"
      xmlns="http://www.w3.org/2000/svg"
      style={style}
      className={className}
    >
      <title>Medium</title>
      <path
        fill={color}
        d="M4.21 0A4.201 4.201 0 0 0 0 4.21v15.58A4.201 4.201 0 0 0 4.21 24h15.58A4.201 4.201 0 0 0 24 19.79v-1.093c-.137.013-.278.02-.422.02-2.577 0-4.027-2.146-4.09-4.832a7.592 7.592 0 0 1 .022-.708c.093-1.186.475-2.241 1.105-3.022a3.885 3.885 0 0 1 1.395-1.1c.468-.237 1.127-.367 1.664-.367h.023c.101 0 .202.004.303.01V4.211A4.201 4.201 0 0 0 19.79 0Zm.198 5.583h4.165l3.588 8.435 3.59-8.435h3.864v.146l-.019.004c-.705.16-1.063.397-1.063 1.254h-.003l.003 10.274c.06.676.424.885 1.063 1.03l.02.004v.145h-4.923v-.145l.019-.005c.639-.144.994-.353 1.054-1.03V7.267l-4.745 11.15h-.261L6.15 7.569v9.445c0 .857.358 1.094 1.063 1.253l.02.004v.147H4.405v-.147l.019-.004c.705-.16 1.065-.397 1.065-1.253V6.987c0-.857-.358-1.094-1.064-1.254l-.018-.004zm19.25 3.668c-1.086.023-1.733 1.323-1.813 3.124H24V9.298a1.378 1.378 0 0 0-.342-.047Zm-1.862 3.632c-.1 1.756.86 3.239 2.204 3.634v-3.634z"
      />
    </svg>
  );
};

export default MediumIcon;

```

## File: connectfy-client/src/assets/icons/TelegramIcon.tsx
```typescript
import { CSSProperties, FC } from "react";

interface IProps {
  style?: CSSProperties;
  className?: string;
  color?: string;
}

const TelegramIcon: FC<IProps> = ({
  style,
  className,
  color = "var(--text-color)",
}) => {
  return (
    <svg
      role="img"
      viewBox="0 0 24 24"
      xmlns="http://www.w3.org/2000/svg"
      style={style}
      className={className}
    >
      <title>Telegram</title>
      <path
        fill={color}
        d="M11.944 0A12 12 0 0 0 0 12a12 12 0 0 0 12 12 12 12 0 0 0 12-12A12 12 0 0 0 12 0a12 12 0 0 0-.056 0zm4.962 7.224c.1-.002.321.023.465.14a.506.506 0 0 1 .171.325c.016.093.036.306.02.472-.18 1.898-.962 6.502-1.36 8.627-.168.9-.499 1.201-.82 1.23-.696.065-1.225-.46-1.9-.902-1.056-.693-1.653-1.124-2.678-1.8-1.185-.78-.417-1.21.258-1.91.177-.184 3.247-2.977 3.307-3.23.007-.032.014-.15-.056-.212s-.174-.041-.249-.024c-.106.024-1.793 1.14-5.061 3.345-.48.33-.913.49-1.302.48-.428-.008-1.252-.241-1.865-.44-.752-.245-1.349-.374-1.297-.789.027-.216.325-.437.893-.663 3.498-1.524 5.83-2.529 6.998-3.014 3.332-1.386 4.025-1.627 4.476-1.635z"
      />
    </svg>
  );
};

export default TelegramIcon;

```

## File: connectfy-client/src/assets/icons/InstagramIcon.tsx
```typescript
import { CSSProperties, FC } from "react";

interface IProps {
  style?: CSSProperties;
  className?: string;
  color?: string;
}

const InstagramIcon: FC<IProps> = ({
  style,
  className,
  color = "var(--text-color)",
}) => {
  return (
    <svg
      role="img"
      viewBox="0 0 24 24"
      xmlns="http://www.w3.org/2000/svg"
      style={style}
      className={className}
    >
      <title>Instagram</title>
      <path
        fill={color}
        d="M7.0301.084c-1.2768.0602-2.1487.264-2.911.5634-.7888.3075-1.4575.72-2.1228 1.3877-.6652.6677-1.075 1.3368-1.3802 2.127-.2954.7638-.4956 1.6365-.552 2.914-.0564 1.2775-.0689 1.6882-.0626 4.947.0062 3.2586.0206 3.6671.0825 4.9473.061 1.2765.264 2.1482.5635 2.9107.308.7889.72 1.4573 1.388 2.1228.6679.6655 1.3365 1.0743 2.1285 1.38.7632.295 1.6361.4961 2.9134.552 1.2773.056 1.6884.069 4.9462.0627 3.2578-.0062 3.668-.0207 4.9478-.0814 1.28-.0607 2.147-.2652 2.9098-.5633.7889-.3086 1.4578-.72 2.1228-1.3881.665-.6682 1.0745-1.3378 1.3795-2.1284.2957-.7632.4966-1.636.552-2.9124.056-1.2809.0692-1.6898.063-4.948-.0063-3.2583-.021-3.6668-.0817-4.9465-.0607-1.2797-.264-2.1487-.5633-2.9117-.3084-.7889-.72-1.4568-1.3876-2.1228C21.2982 1.33 20.628.9208 19.8378.6165 19.074.321 18.2017.1197 16.9244.0645 15.6471.0093 15.236-.005 11.977.0014 8.718.0076 8.31.0215 7.0301.0839m.1402 21.6932c-1.17-.0509-1.8053-.2453-2.2287-.408-.5606-.216-.96-.4771-1.3819-.895-.422-.4178-.6811-.8186-.9-1.378-.1644-.4234-.3624-1.058-.4171-2.228-.0595-1.2645-.072-1.6442-.079-4.848-.007-3.2037.0053-3.583.0607-4.848.05-1.169.2456-1.805.408-2.2282.216-.5613.4762-.96.895-1.3816.4188-.4217.8184-.6814 1.3783-.9003.423-.1651 1.0575-.3614 2.227-.4171 1.2655-.06 1.6447-.072 4.848-.079 3.2033-.007 3.5835.005 4.8495.0608 1.169.0508 1.8053.2445 2.228.408.5608.216.96.4754 1.3816.895.4217.4194.6816.8176.9005 1.3787.1653.4217.3617 1.056.4169 2.2263.0602 1.2655.0739 1.645.0796 4.848.0058 3.203-.0055 3.5834-.061 4.848-.051 1.17-.245 1.8055-.408 2.2294-.216.5604-.4763.96-.8954 1.3814-.419.4215-.8181.6811-1.3783.9-.4224.1649-1.0577.3617-2.2262.4174-1.2656.0595-1.6448.072-4.8493.079-3.2045.007-3.5825-.006-4.848-.0608M16.953 5.5864A1.44 1.44 0 1 0 18.39 4.144a1.44 1.44 0 0 0-1.437 1.4424M5.8385 12.012c.0067 3.4032 2.7706 6.1557 6.173 6.1493 3.4026-.0065 6.157-2.7701 6.1506-6.1733-.0065-3.4032-2.771-6.1565-6.174-6.1498-3.403.0067-6.156 2.771-6.1496 6.1738M8 12.0077a4 4 0 1 1 4.008 3.9921A3.9996 3.9996 0 0 1 8 12.0077"
      />
    </svg>
  );
};

export default InstagramIcon;

```

## File: connectfy-client/src/context/UserContext.tsx
```typescript
import { createContext, useContext, ReactNode } from "react";
import { useGetMeQuery } from "@/modules/profile/api/api";
import { useAuthStore } from "@/store/zustand/useAuthStore";
import { IMe } from "@/modules/profile/types/types";
import { PROVIDER } from "@/common/enums/enums";

interface IUserContext {
  user: IMe | undefined;
  isLoading: boolean;
  isSuccess: boolean;
  isError: boolean;
  usesPasswordAuth: boolean;
  usesOAuth: boolean;
  hasPhoneNumber: boolean;
}

const UserContext = createContext<IUserContext | undefined>(undefined);

export function UserProvider({ children }: { children: ReactNode }) {
  const { access_token } = useAuthStore();

  // ✅ Bütün app üçün YALNIZ BİR dəfə subscribe olur
  const result = useGetMeQuery(undefined, { skip: !access_token });

  const user = result.data;

  const value: IUserContext = {
    user,
    isLoading: result.isLoading,
    isSuccess: result.isSuccess,
    isError: result.isError,
    usesPasswordAuth: user?.provider === PROVIDER.PASSWORD,
    usesOAuth: user?.provider === PROVIDER.GOOGLE,
    hasPhoneNumber: !!user?.phoneNumber?.fullPhoneNumber,
  };

  return <UserContext.Provider value={value}>{children}</UserContext.Provider>;
}

// Hook — artıq API çağırmır, sadəcə context-dən oxuyur
export function useUser() {
  const ctx = useContext(UserContext);
  if (!ctx) throw new Error("useUser must be used within UserProvider");
  return ctx;
}

```

## File: connectfy-client/src/context/ThemeContext.tsx
```typescript
import { LOCAL_STORAGE_KEYS, THEME } from "@/common/enums/enums";
import React, { createContext, useContext, useEffect, useState } from "react";
import { useSelector } from "react-redux";
import { generalSettingsApi } from "@/modules/settings/GeneralSettings/api/api";

interface ThemeContextType {
  theme: THEME;
  toggleTheme: (theme: THEME) => void;
}

export const ThemeContext = createContext<ThemeContextType | undefined>(
  undefined,
);

export const ThemeProvider = ({ children }: { children: React.ReactNode }) => {
  // Redux-dan API theme-i oxu (login olduqdan sonra gəlir)
  const apiTheme = useSelector(
    (state: any) =>
      generalSettingsApi.endpoints.getGeneralSettings.select(undefined)(state)
        ?.data?.theme as THEME | undefined,
  );

  // LocalStorage-dan fallback — API gəlməmişdən əvvəl işləyir (login yoxdur vs.)
  const [activeTheme, setActiveTheme] = useState<THEME>(
    () =>
      (localStorage.getItem(LOCAL_STORAGE_KEYS.APP_THEME) as THEME) ||
      THEME.LIGHT,
  );

  // API theme gəldikdə local state-i sinxronlaşdır
  useEffect(() => {
    if (apiTheme && apiTheme !== activeTheme) {
      setActiveTheme(apiTheme);
      localStorage.setItem(LOCAL_STORAGE_KEYS.APP_THEME, apiTheme);
    }
  }, [apiTheme]);

  useEffect(() => {
    const applyTheme = (targetTheme: THEME) => {
      const actual =
        targetTheme === THEME.DEVICE
          ? window.matchMedia("(prefers-color-scheme: dark)").matches
            ? THEME.DARK
            : THEME.LIGHT
          : targetTheme;

      document.documentElement.setAttribute("data-theme", actual.toLowerCase());
    };

    applyTheme(activeTheme);

    if (activeTheme === THEME.DEVICE) {
      const mediaQuery = window.matchMedia("(prefers-color-scheme: dark)");
      const listener = () => applyTheme(THEME.DEVICE);
      mediaQuery.addEventListener("change", listener);
      return () => mediaQuery.removeEventListener("change", listener);
    }
  }, [activeTheme]);

  // toggleTheme artıq local state-i yeniləyir
  // API update-i settings səhifəsindəki mutation həll edir
  const toggleTheme = (newTheme: THEME) => {
    setActiveTheme(newTheme); // UI anında reaksiya verir
    localStorage.setItem(LOCAL_STORAGE_KEYS.APP_THEME, newTheme);
  };

  return (
    <ThemeContext.Provider value={{ theme: activeTheme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
};

export const useTheme = () => {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error("useTheme must be used within ThemeProvider");
  return ctx;
};

```

## File: connectfy-client/src/context/ContextMenuContext.tsx
```typescript
import ContextMenuRenderer from "@/components/ContextMenu/ContextMenuRenderer";
import { createContext, useState, useCallback, ReactNode } from "react";

interface ContextMenuState {
  x: number;
  y: number;
  content: ReactNode;
  isOpen: boolean;
}

interface ContextMenuContextType {
  openMenu: (x: number, y: number, content: ReactNode) => void;
  closeMenu: () => void;
}

export const ContextMenuContext = createContext<ContextMenuContextType | null>(
  null,
);

export const ContextMenuProvider = ({ children }: { children: ReactNode }) => {
  const [state, setState] = useState<ContextMenuState>({
    x: 0,
    y: 0,
    content: null,
    isOpen: false,
  });

  const openMenu = useCallback((x: number, y: number, content: ReactNode) => {
    setState({ x, y, content, isOpen: true });
  }, []);

  const closeMenu = useCallback(() => {
    setState((prev) => ({ ...prev, isOpen: false }));
  }, []);

  return (
    <ContextMenuContext.Provider value={{ openMenu, closeMenu }}>
      {children}
      <ContextMenuRenderer {...state} onClose={closeMenu} />
    </ContextMenuContext.Provider>
  );
};

```

## File: connectfy-client/src/context/SocketContext.tsx
```typescript
import {
  createContext,
  ReactNode,
  useContext,
  useEffect,
  useState,
} from "react";
import { io, Socket } from "socket.io-client";
import { baseUrl } from "@/common/constants/constants";
import { useAuthStore } from "@/store/zustand/useAuthStore";

interface AppSockets {
  main: Socket | null;
  notification: Socket | null;
}

const SocketsContext = createContext<AppSockets>({
  main: null,
  notification: null,
});

export const SockerProvider = ({ children }: { children: ReactNode }) => {
  const [sockets, setSockets] = useState<AppSockets>({
    main: null,
    notification: null,
  });

  const { access_token } = useAuthStore();

  useEffect(() => {
    if (!access_token) {
      setSockets({ main: null, notification: null });
      return;
    }

    const socketOptions = {
      // The NestJS gateways expect `handshake.auth.token` in `Bearer <jwt>` format.
      auth: {
        token: `Bearer ${access_token}`,
        access_token,
      },
      transports: ["websocket", "polling"],
    };
    const mainSocket = io(baseUrl, socketOptions);
    const notificationSocket = io(`${baseUrl}/notification`, socketOptions);

    setSockets({
      main: mainSocket,
      notification: notificationSocket,
    });

    return () => {
      mainSocket.disconnect();
      notificationSocket.disconnect();
    };
  }, [access_token]);

  return (
    <SocketsContext.Provider value={sockets}>
      {children}
    </SocketsContext.Provider>
  );
};

export const useSockets = () => useContext(SocketsContext);

```

## File: connectfy-client/src/common/enums/enums.ts
```typescript
export enum REDUCER_PATH {
  PROFILE = "profilePath",
  AUTH = "authPath",
  GENERAL_SETTINGS = "generalSettingsPath",
  NOTIFICATIONS_SETTINGS = "notificationsSettingsPath",
  PRIVACY_SETTINGS = "privacySettingsPath",
}

export enum TAG_TYPES {
  PROFILE = "Profile",
  AUTH = "Auth",
  GENERAL_SETTINGS = "GeneralSettings",
  NOTIFICATIONS_SETTINGS = "NotificationsSettings",
  PRIVACY_SETTINGS = "PrivacySettings",
}

export enum LANGUAGE {
  EN = "en",
  AZ = "az",
  RU = "ru",
  TR = "tr",
}

export enum PROVIDER {
  PASSWORD = "PASSWORD",
  GOOGLE = "GOOGLE",
  FACEBOOK = "FACEBOOK",
}

export enum RESOURCE {
  AUTH = "auth",
  GENERAL_SETTINGS = "general-settings",
  PRIVACY_SETTINGS = "privacy-settings",
  NOTIFICATION_SETTINGS = "notification-settings",
  USER = "user",
  PROFILE = "profile",
  ACCOUNT_SETTINGS = "account-settings",
  ALL_USERS = "all-users",
  USER_PROFİLE = "user-profile",
  MY_FRIENDS = "my-friends",
  NOTIFICATIONS = "notifications",
  BLOCKLIST = "blocklist",
}

export enum GENDER {
  MALE = "MALE",
  FEMALE = "FEMALE",
  OTHER = "OTHER",
}

export enum IDENTIFIER_TYPE {
  USERNAME = "USERNAME",
  EMAIL = "EMAIL",
  PHONE_NUMBER = "PHONE_NUMBER",
}

export enum FORGOT_PASSWORD_IDENTIFIER_TYPE {
  EMAIL = "EMAIL",
  PHONE_NUMBER = "PHONE_NUMBER",
}

export enum TOKEN_TYPE {
  PASSWORD_RESET = "PASSWORD_RESET",
  DELETE_ACCOUNT = "DELETE_ACCOUNT",
  RESTORE_ACCOUNT = "RESTORE_ACCOUNT",
  CHANGE_USERNAME = "CHANGE_USERNAME",
  CHANGE_EMAIL = "CHANGE_EMAIL",
  CHANGE_PASSWORD = "CHANGE_PASSWORD",
  CHANGE_PHONE_NUMBER = "CHANGE_PHONE_NUMBER",
  DEACTIVATE_ACCOUNT = "DEACTIVATE_ACCOUNT",
  TWO_FACTOR = "TWO_FACTOR",
}

export enum TWO_FACTOR_ACTION {
  ENABLE = "ENABLE",
  DISABLE = "DISABLE",
}

export enum PHONE_NUMBER_ACTION {
  UPDATE = "UPDATE",
  REMOVE = "REMOVE",
}

export enum PRIVACY_SETTINGS_CHOICE {
  EVERYONE = "EVERYONE",
  MY_FRIENDS = "MY_FRIENDS",
  NOBODY = "NOBODY",
}

export enum THEME {
  DARK = "DARK",
  LIGHT = "LIGHT",
  DEVICE = "DEVICE",
}

export enum NOTIFICATION_SOUND_MODE {
  SOUND = "SOUND",
  SILENT = "SILENT",
  DND = "DND",
}

export enum NOTIFICATION_CONTENT_MODE {
  HEADER_AND_CONTENT = "HEADER_AND_CONTENT",
  HEADER_ONLY = "HEADER_ONLY",
  HIDE_NOTIFICATION = "HIDE_NOTIFICATION",
}

export enum STARTUP_PAGE {
  MESSENGER = "MESSENGER",
  GROUPS = "GROUPS",
  CHANNELS = "CHANNELS",
  USERS = "USERS",
  NOTIFICATIONS = "NOTIFICATIONS",
  PROFILE = "PROFILE",
}

export enum TIME_FORMAT {
  H24 = "24h",
  H12 = "12h",
}

export enum DATE_FORMAT {
  DDMMYYYY = "DD/MM/YYYY",
  MMDDYYYY = "MM/DD/YYYY",
}

export enum DELETE_REASON {
  USER_REQUEST = "USER_REQUEST",
  TERMS_VIOLATION = "TERMS_VIOLATION",
  SPAM = "SPAM",
  FRAUD = "FRAUD",
  SECURITY = "SECURITY",
  INACTIVITY = "INACTIVITY",
  ADMIN_ACTION = "ADMIN_ACTION",
  OTHER = "OTHER",
}

export enum SOCIAL_LINK_PLATFORM {
  INSTAGRAM = "Instagram",
  FACEBOOK = "Facebook",
  X = "X",
  LINKEDIN = "LinkedIn",
  YOUTUBE = "Youtube",
  GITHUB = "Github",
  REDDIT = "Reddit",
  PINTEREST = "Pinterest",
  TIKTOK = "Tiktok",
  TELEGRAM = "Telegram",
  DISCORD = "Discord",
  SNAPCHAT = "Snapchat",
  WHATSAPP = "Whatsapp",
  MEDIUM = "Medium",
  BEHANCE = "Behance",
  EMAIL = "Email",
  WEBSITE = "Website",
  OTHER = "Other",
}

export enum LOCAL_STORAGE_KEYS {
  LANG = "lang",
  ACCESS_TOKEN = "access_token",
  DEVICE_ID = "deviceId",
  APP_THEME = "app-theme",
  AUTH_PAGE = "auth-page",
  LOGIN_MODE = "login-mode",
  FORGOT_PASSWORD_MODE = "forgot-password-mode",
  OTP_EXPIRES_AT = "otp_expires_at",
  SIGNUP_FORM = "signup_form",
}

export enum TIME_DIFFERENCE_TYPE {
  NOW = "just_now",
  SECOND = "seconds_ago",
  MINUTE = "minutes_ago",
  HOUR = "hours_ago",
  DAY = "days_ago",
  WEEK = "weeks_ago",
  MONTH = "months_ago",
  YEAR = "years_ago",
}

export enum CHECK_UNIQUE_FIELD {
  USERNAME = "username",
  EMAIL = "email",
  PHONE_NUMBER = "phone_number",
}

export enum ModalView {
  SELECTION = "SELECTION",
  FORM = "FORM",
}

export enum DELETE_REASON_CODE {
  NOT_USEFUL = "NOT_USEFUL",
  PRIVACY_CONCERNS = "PRIVACY_CONCERNS",
  FOUND_ALTERNATIVE = "FOUND_ALTERNATIVE",
  TECHNICAL_ISSUES = "TECHNICAL_ISSUES",
  OTHER = "OTHER",
}

export enum ProfilePhotoUpdateAction {
  Remove = "Remove",
  Update = "Update",
  SetDefault = "SetDefault",
}

export enum AvatarFormats {
  Adventurer = "adventurer",
  Avataaars = "avataaars",
  Personas = "personas",
  ToonHead = "toon-head",
  Micah = "micah",
}

export enum FriendshipStatus {
  Pending = "PENDING",
  Accepted = "ACCEPTED",
  Blocked = "BLOCKED",
}

export enum FriendshipRequestType {
  Received = "Received",
  Sent = "Sent",
}

export enum NotificationType {
  // Social
  FRIENDSHIP_REQUEST_SENT = "friendship_request_sent",
  FRIENDSHIP_REQUEST_ACCEPTED = "friendship_request_accepted",
  FRIENDSHIP_REQUEST_DECLINED = "friendship_request_declined",
  // Messaging (Phase 2)
  NEW_MESSAGE = "message_new",
  // System
  SYSTEM_ALERT = "system_alert",
  ACCOUNT_WARNING = "account_warning",
}

export enum NotificationStatus {
  Unread = "unread",
  Read = "read",
  Archived = "archived",
  Accepted = "accepted",
  Declined = "declined",
  Pending = "pending",
}

export enum NotificationChannel {
  InApp = "in_app",
  Push = "push",
  Both = "both",
}

export enum Push_DELIVERY_STATUS {
  PENDING = "pending",
  DELIVERED = "delivered",
  FAILED = "failed",
  SKIPPED = "skipped", // user preference disabled
}

export enum NotificationResource {
  Friendship = "friendship",
  Message = "message",
  System = "system",
  Account = "account",
}

```

## File: connectfy-client/src/common/hooks/usePresenceHeartbeat.ts
```typescript
import { useEffect } from 'react';
import api from '@/common/api/axios';
import { API_ENDPOINTS } from '@/common/constants/apiEndpoints';
import { useAuthStore } from '@/store/zustand/useAuthStore';

export const usePresenceHeartbeat = () => {
  const access_token = useAuthStore((state) => state.access_token);

  useEffect(() => {
    if (!access_token) return;

    const sendHeartbeat = async () => {
      if (document.visibilityState === 'visible') {
        try {
          const res = await api.post(API_ENDPOINTS.USER.HEARTBEAT);
          console.info('[Presence] Heartbeat sent successfully', res.status);
        } catch (error: any) {
          console.info(
            '[Presence] Heartbeat failed',
            error?.response?.status ?? error?.message,
          );
        }
      }
    };

    // Initial heartbeat
    sendHeartbeat();

    const intervalId = setInterval(sendHeartbeat, 25000);

    const handleVisibilityChange = () => {
      if (document.visibilityState === 'visible') {
        sendHeartbeat();
      }
    };

    document.addEventListener('visibilitychange', handleVisibilityChange);

    return () => {
      clearInterval(intervalId);
      document.removeEventListener('visibilitychange', handleVisibilityChange);
    };
  }, [access_token]);
};

```

## File: connectfy-client/src/common/constants/routet.ts
```typescript
export const ROUTER = {
  MAIN: "/",
  TERMS_AND_CONDITIONS: "/terms-and-conditions",
  AUTH: {
    MAIN: "/auth",
    LOGIN: "/auth/login",
    VERIFY_LOGIN: "/auth/login/verify",
    SIGNUP: "/auth/signup",
    VERIFY_SIGNUP: "/auth/signup/verify",
    FORGOT_PASSWORD: "/auth/forgot-password",
    RESET_PASSWORD: "/auth/reset-password",
  },
  MESSENGER: {
    MAIN: "/messenger",
  },
  GROUPS: {
    MAIN: "/groups",
  },
  CHANNELS: {
    MAIN: "/channels",
  },
  USERS: {
    MAIN: "/users",
    PROFILE: "/users/profile",
    BLOCKLIST: "/users/blocklist",
    FRIENDS: "/users/friends",
    REQUESTS: "/users/requests",
    SEARCH: "/users/search",
  },
  NOTIFICATIONS: {
    MAIN: "/notifications",
  },
  PROFILE: {
    MAIN: "/profile",
  },
  SETTINGS: {
    MAIN: "/settings",
    GENERAL: "/settings/general",
    PRIVACY: "/settings/privacy",
    ACCOUNT: "/settings/account",
    BACKGROUND: "/settings/background",
    NOTIFICATION: "/settings/notification",
    SHORTCUT: "/settings/shortcut",
  },
};

```

## File: connectfy-client/src/common/constants/constants.ts
```typescript
import { ICountry } from "../interfaces/interfaces";

export const baseUrl = `http://${import.meta.env.VITE_IP_ADDRESS}:${import.meta.env.VITE_PORT}`;

export const COUNTRIES: ICountry[] = [
  {
    key: "az",
    name: "azerbaijan",
    flag: "fi fi-az",
    code: "+994",
    numberLength: 9,
    totalLength: 12,
    format: "00 000 00 00",
  },
  {
    key: "tr",
    name: "türkiye",
    flag: "fi fi-tr",
    code: "+90",
    numberLength: 10,
    totalLength: 13,
    format: "000 000 00 00",
  },
  {
    key: "ru",
    name: "russia",
    flag: "fi fi-ru",
    code: "+7",
    numberLength: 10,
    totalLength: 13,
    format: "000 000 00 00",
  },
  {
    key: "us",
    name: "usa",
    flag: "fi fi-us",
    code: "+1",
    numberLength: 10,
    totalLength: 12,
    format: "000 000 0000",
  },
  {
    key: "gb",
    name: "uk",
    flag: "fi fi-gb",
    code: "+44",
    numberLength: 10,
    totalLength: 12,
    format: "0000 000 000",
  },
  {
    key: "de",
    name: "germany",
    flag: "fi fi-de",
    code: "+49",
    numberLength: 10,
    totalLength: 11,
    format: "000 0000000",
  },
  {
    key: "fr",
    name: "france",
    flag: "fi fi-fr",
    code: "+33",
    numberLength: 9,
    totalLength: 13, // "0 00 00 00 00"
    format: "0 00 00 00 00",
  },
  {
    key: "es",
    name: "spain",
    flag: "fi fi-es",
    code: "+34",
    numberLength: 9,
    totalLength: 11,
    format: "000 000 000",
  },
  {
    key: "it",
    name: "italy",
    flag: "fi fi-it",
    code: "+39",
    numberLength: 10,
    totalLength: 12,
    format: "000 000 0000",
  },
  {
    key: "cn",
    name: "china",
    flag: "fi fi-cn",
    code: "+86",
    numberLength: 11,
    totalLength: 13,
    format: "000 0000 0000",
  },
  {
    key: "jp",
    name: "japan",
    flag: "fi fi-jp",
    code: "+81",
    numberLength: 10,
    totalLength: 12,
    format: "00 0000 0000",
  },
  {
    key: "kr",
    name: "south_korea",
    flag: "fi fi-kr",
    code: "+82",
    numberLength: 10,
    totalLength: 12,
    format: "00 0000 0000",
  },
  {
    key: "ua",
    name: "ukraine",
    flag: "fi fi-ua",
    code: "+380",
    numberLength: 9,
    totalLength: 12,
    format: "00 000 00 00",
  },
  {
    key: "ge",
    name: "georgia",
    flag: "fi fi-ge",
    code: "+995",
    numberLength: 9,
    totalLength: 12,
    format: "000 00 00 00",
  },
  {
    key: "ir",
    name: "iran",
    flag: "fi fi-ir",
    code: "+98",
    numberLength: 10,
    totalLength: 12,
    format: "000 000 0000",
  },
  {
    key: "in",
    name: "india",
    flag: "fi fi-in",
    code: "+91",
    numberLength: 10,
    totalLength: 11,
    format: "00000 00000",
  },
  {
    key: "ca",
    name: "canada",
    flag: "fi fi-ca",
    code: "+1",
    numberLength: 10,
    totalLength: 12,
    format: "000 000 0000",
  },
  {
    key: "kz",
    name: "kazakhstan",
    flag: "fi fi-kz",
    code: "+7",
    numberLength: 10,
    totalLength: 13,
    format: "000 000 00 00",
  },
  {
    key: "uz",
    name: "uzbekistan",
    flag: "fi fi-uz",
    code: "+998",
    numberLength: 9,
    totalLength: 12,
    format: "00 000 00 00",
  },
  {
    key: "ae",
    name: "uae",
    flag: "fi fi-ae",
    code: "+971",
    numberLength: 9,
    totalLength: 11,
    format: "0 000 0000",
  },
];

```

## File: connectfy-client/src/common/constants/apiEndpoints.ts
```typescript
export const API_ENDPOINTS = {
  AUTH: {
    LOGIN: "/auth/login",
    SIGNUP: "/auth/signup",
    LOGIN_VERIFY: "/auth/login/verify",
    SIGNUP_VERIFY: "/auth/signup/verify",
    FORGOT_PASSWORD: "/auth/forgot-password",
    RESET_PASSWORD: "/auth/reset-password",
    LOGOUT: "/auth/logout",
    GOOGLE_LOGIN: "/auth/google/login",
    GOOGLE_SIGNUP: "/auth/google/signup",
    FACE_DESCRIPTOR: "/auth/face-descriptor",
    IS_VALID_TOKEN: "/auth/is-valid-token",
    REFRESH: "/auth/refresh",
    AUTHENTICATE_USER: "/auth/authenticate-user",
    RESTORE_ACCOUNT: "/auth/restore-account",
    RESEND_SIGNUP_VERIFY: "/auth/signup/verify/resend",
  },
  USER: {
    ME: "/user/me",
    CHANGE_USERNAME: "/user/change-username",
    CHANGE_EMAIL: "/user/change-email",
    VERIFY_CHANGE_EMAIL: "/user/change-email/verify",
    CHANGE_PASSWORD: "/user/change-password",
    CHANGE_PHONE_NUMBER: "/user/change-phone-number",
    CHECK_UNIQUE: "/user/check-unique",
    UPDATE_TWO_FACTOR: "/user/two-factor",
    DELETE_ACCOUNT: "/user/delete-account",
    DEACTIVATE_ACCOUNT: "/user/deactivate-account",
    HEARTBEAT: "/user/heartbeat",
  },
  ACCOUNT: {
    PROFILE: {
      GET: "/account/profile/get",
      FIND_ONE: (userId: string) =>
        `/account/profile/findOneByUserId/${userId}`,
      UPDATE: "/account/profile/update",
      UPDATE_AVATAR: "/account/profile/update-avatar",
      UPDATE_DEFAULT_AVATAR: "/account/profile/update-default-avatar",
      SEARCH: "/account/profile/search",
    },
    SETTINGS: {
      GENERAL_SETTINGS: {
        GET: "/account/settings/general-settings/get",
        UPDATE: "/account/settings/general-settings/update",
        RESET: "/account/settings/general-settings/reset",
      },
      NOTIFICATION_SETTINGS: {
        GET: "/account/settings/notification-settings/get",
        UPDATE: "/account/settings/notification-settings/update",
      },
      PRIVACY_SETTINGS: {
        GET: "/account/settings/privacy-settings/get",
        UPDATE: "/account/settings/privacy-settings/update",
      },
    },
    SOCIAL_LINK: {
      GET: "/account/social-link/get",
      CREATE: "/account/social-link/create",
      UPDATE: "/account/social-link/update",
      UPDATE_RANK: "/account/social-link/update-rank",
      REMOVE: "/account/social-link/remove",
      REMOVE_MANY: "/account/social-link/removeMany",
    },
  },
  FILES: {
    PRESIGNED_UPLOAD: "/files/presigned-upload",
  },
  RELATIONSHIP: {
    FRIENDSHIP: {
      SEND_REQUEST: "/relationship/friendship/send-request",
      ACCEPT_REQUEST: "/relationship/friendship/accept-request",
      DECLINE_REQUEST: "/relationship/friendship/decline-request",
      CANCEL_REQUEST: "/relationship/friendship/cancel-request",
      UNFRIEND: "/relationship/friendship/unfriend",
      UPDATE_CLOSE_FRIEND: "/relationship/friendship/update-close-friend",
      UPDATE_NOTIFICATION: "/relationship/friendship/update-notification",
      FIND_FRIENDS: "/relationship/friendship/find-friends",
      FIND_REQUESTS: "/relationship/friendship/find-requests",
      REQUESTS_COUNT: "/relationship/friendship/requests-count",
      FIND_ONLINE_FRIENDS: "/relationship/friendship/find-online-friends",
      FIND_SUGGESTIONS: "/relationship/friendship/find-suggestions",
      FIND_MUTUAL_FRIENDS: "/relationship/friendship/find-mutual-friends",
    },
    BLOCKLIST: {
      FIND_BLOCKED: "/relationship/blocklist/find-blocked",
      UNBLOCK: "/relationship/blocklist/unblock",
      BLOCK: "/relationship/blocklist/block",
    },
  },
  NOTIFICATION_ACTION_HISTORY: {
    NOTIFICATION: {
      GET_ALL: "/notification-action-history/notification/all",
      COUNT_UNREAD: "/notification-action-history/notification/countUnread",
      MARK_READ: "/notification-action-history/notification/markRead",
      MARK_UNREAD: "/notification-action-history/notification/markUnread",
      MARK_ALL_READ: "/notification-action-history/notification/markAllRead",
      REMOVE: "/notification-action-history/notification/remove",
      REMOVE_ALL: "/notification-action-history/notification/removeAll",
    },
  },
};

```

## File: connectfy-client/src/common/utils/keyPressDown.ts
```typescript
import React from "react";

export function onPressEnter(
  e: React.KeyboardEvent<HTMLElement>,
  onPress: () => void
) {
  if (e.key === "Enter") {
    e.preventDefault();
    onPress();
  }
}

export function onPressEsc(
  e: React.KeyboardEvent<HTMLElement>,
  onPress: () => void
) {
  if (e.key === "Escape") {
    e.preventDefault();
    onPress();
  }
}

// ============ DİGƏR FAYDALI KEY DƏYƏRLƏRI ============
/*
e.key === "Enter"       // ✅ Enter düyməsi
e.key === "Escape"      // ✅ Esc düyməsi
e.key === " "           // ✅ Space düyməsi
e.key === "Tab"         // ✅ Tab düyməsi
e.key === "Backspace"   // ✅ Backspace düyməsi
e.key === "Delete"      // ✅ Delete düyməsi
e.key === "ArrowUp"     // ✅ Yuxarı ox
e.key === "ArrowDown"   // ✅ Aşağı ox
e.key === "ArrowLeft"   // ✅ Sol ox
e.key === "ArrowRight"  // ✅ Sağ ox
*/
```

## File: connectfy-client/src/common/utils/toast.ts
```typescript
import { useIsMobile } from "@/hooks/useIsMobile";
import { SnackbarOrigin, VariantType, enqueueSnackbar } from "notistack";

type SnackOptions = {
  variant?: VariantType;
  autoHideDuration?: number;
  anchorOrigin?: SnackbarOrigin;
  size?: "small" | "medium" | "large";
};

const isMobile = useIsMobile();

const getAnchorOrigin = (customOrigin?: SnackbarOrigin): SnackbarOrigin => {
  if (customOrigin && !isMobile) {
    return customOrigin;
  }

  return isMobile
    ? { vertical: "bottom", horizontal: "center" }
    : { vertical: "bottom", horizontal: "right" };
};

export const snack = {
  show: (message: string, options: SnackOptions = {}) => {
    const size = options.size ?? "large";

    enqueueSnackbar(message, {
      variant: options.variant ?? "default",
      autoHideDuration: options.autoHideDuration ?? 4000,
      anchorOrigin: getAnchorOrigin(options.anchorOrigin),
      className: `snackbar-${size}`,
    });
  },

  success: (message: string, options: Omit<SnackOptions, "variant"> = {}) => {
    snack.show(message, { ...options, variant: "success" });
  },

  error: (message: string, options: Omit<SnackOptions, "variant"> = {}) => {
    snack.show(message, { ...options, variant: "error" });
  },

  warning: (message: string, options: Omit<SnackOptions, "variant"> = {}) => {
    snack.show(message, { ...options, variant: "warning" });
  },

  info: (message: string, options: Omit<SnackOptions, "variant"> = {}) => {
    snack.show(message, { ...options, variant: "info" });
  },
};

```

## File: connectfy-client/src/common/utils/notificationHelpers.ts
```typescript
import { NOTIFICATION_CONTENT_MODE } from "@/common/enums/enums";
import { INotificationSettings } from "@/modules/settings/NotificationSettings/types/types";

export const resolveToastContent = (
  settings: INotificationSettings | undefined | null,
): { showToast: boolean; showBody: boolean } => {
  if (!settings) {
    return { showToast: true, showBody: true };
  }

  switch (settings.notificationContentMode) {
    case NOTIFICATION_CONTENT_MODE.HIDE_NOTIFICATION:
      return { showToast: false, showBody: false };
    case NOTIFICATION_CONTENT_MODE.HEADER_ONLY:
      return { showToast: true, showBody: false };
    case NOTIFICATION_CONTENT_MODE.HEADER_AND_CONTENT:
    default:
      return { showToast: true, showBody: true };
  }
};

```

## File: connectfy-client/src/common/utils/keyboard.ts
```typescript
export const isMac = () => {
  return (
    typeof window !== "undefined" &&
    /Mac|iPod|iPhone|iPad/.test(navigator.platform)
  );
};

export const getModifierKey = () => (isMac() ? "⌘" : "Ctrl");
export const getAltKey = () => (isMac() ? "⌥" : "Alt");
export const getShiftKey = () => "Shift";

```

## File: connectfy-client/src/common/utils/checkValues.ts
```typescript
import { v4 as uuid, validate } from "uuid";
import { LOCAL_STORAGE_KEYS } from "../enums/enums";

export function checkEmptyString(value: string): boolean {
  return value.trim() !== "";
}

export function checkDeviceId(): string {
  const deviceId = localStorage.getItem(LOCAL_STORAGE_KEYS.DEVICE_ID);

  if (deviceId && validate(deviceId)) return deviceId;

  const newDeviceId = uuid();
  localStorage.setItem(LOCAL_STORAGE_KEYS.DEVICE_ID, newDeviceId);
  return newDeviceId;
}

export function checkUrl(url: string): boolean {
  try {
    new URL(url);
    return true;
  } catch {
    return false;
  }
}

```

## File: connectfy-client/src/common/utils/snackManager.ts
```typescript
import toast, { ToastPosition } from "react-hot-toast";
import CustomToast from "@/components/ui/CustomToast/CustomToast";

type ToastType = "success" | "error" | "warning" | "info" | "default";
type SnackOptions = {
  duration?: number;
  position?: ToastPosition;
};

const isMobile = () => window.innerWidth < 1024;
const getPosition = (position?: ToastPosition) => {
  if (position) return position;
  return isMobile() ? "bottom-center" : "bottom-right";
};

const show = (message: string, type: ToastType, options: SnackOptions = {}) => {
  toast.custom(
    (t) =>
      CustomToast({
        toastId: t.id,
        message,
        type,
        visible: t.visible,
      }),
    {
      duration: options.duration ?? 3000,
      position: getPosition(options.position),
    },
  );
};

export const snack = {
  show: (message: string, options?: SnackOptions) =>
    show(message, "default", options),
  success: (message: string, options?: SnackOptions) =>
    show(message, "success", options),
  error: (message: string, options?: SnackOptions) =>
    show(message, "error", options),
  warning: (message: string, options?: SnackOptions) =>
    show(message, "warning", options),
  info: (message: string, options?: SnackOptions) =>
    show(message, "info", options),
};

```

## File: connectfy-client/src/common/utils/cropImage.ts
```typescript
export const createImage = (url: string): Promise<HTMLImageElement> =>
  new Promise((resolve, reject) => {
    const image = new Image();
    image.addEventListener("load", () => resolve(image));
    image.addEventListener("error", (error) => reject(error));
    image.setAttribute("crossOrigin", "anonymous");
    image.src = url;
  });

export async function getCroppedImg(
  imageSrc: string,
  pixelCrop: { x: number; y: number; width: number; height: number },
  fileName: string = "avatar.jpeg",
): Promise<File | null> {
  const image = await createImage(imageSrc);
  const canvas = document.createElement("canvas");
  const ctx = canvas.getContext("2d");

  if (!ctx) return null;

  canvas.width = pixelCrop.width;
  canvas.height = pixelCrop.height;

  ctx.drawImage(
    image,
    pixelCrop.x,
    pixelCrop.y,
    pixelCrop.width,
    pixelCrop.height,
    0,
    0,
    pixelCrop.width,
    pixelCrop.height,
  );

  return new Promise((resolve, reject) => {
    canvas.toBlob((blob) => {
      if (!blob) {
        reject(new Error("Canvas is empty"));
        return;
      }
      const file = new File([blob], fileName, {
        type: "image/jpeg",
        lastModified: Date.now(),
      });
      resolve(file);
    }, "image/jpeg");
  });
}

```

## File: connectfy-client/src/common/utils/skeleton.ts
```typescript
export { Skeleton, SkeletonCard } from "../../components/Skeleton/Skeleton";
export type { SkeletonProps } from "../../components/Skeleton/Skeleton";
export { SettingsSkeleton } from "../../components/Skeleton/settings/SettingsSkeleton";

```

## File: connectfy-client/src/common/utils/routes.ts
```typescript
import { ROUTER } from "@/common/constants/routet";
import { STARTUP_PAGE } from "@/common/enums/enums";

export function getHomeRouteByStartup(startupPage?: STARTUP_PAGE) {
  switch (startupPage) {
    case STARTUP_PAGE.MESSENGER:
      return ROUTER.MESSENGER.MAIN;
    case STARTUP_PAGE.GROUPS:
      return ROUTER.GROUPS.MAIN;
    case STARTUP_PAGE.CHANNELS:
      return ROUTER.CHANNELS.MAIN;
    case STARTUP_PAGE.USERS:
      return ROUTER.USERS.MAIN;
    case STARTUP_PAGE.NOTIFICATIONS:
      return ROUTER.NOTIFICATIONS.MAIN;
    case STARTUP_PAGE.PROFILE:
      return ROUTER.PROFILE.MAIN;
    default:
      return ROUTER.MESSENGER.MAIN;
  }
}

```

## File: connectfy-client/src/common/utils/formatValues.ts
```typescript
import { DATE_FORMAT, TIME_FORMAT } from "@/common/enums/enums";
import dayjs from "dayjs";
import { t } from "i18next";

// ====================
// DD MMMM YYYY
// ====================
export function DDMMMMYYYY(date: string | Date) {
  const d = dayjs(date);
  const day = d.format("DD");
  let monthKey = d.format("MMMM").toLowerCase();
  if (monthKey === "may") monthKey = "may_full";
  const month = t(`calendar.months.${monthKey}`);
  const year = d.format("YYYY");
  return `${day} ${month} ${year}`;
}

// ====================
// DD MMM YYYY
// ====================
export function DDMMMYYYY(date: string | Date) {
  const d = dayjs(date);
  const day = d.format("DD");
  let monthKey = d.format("MMM").toLowerCase();
  if (monthKey === "may") monthKey = "may_full";
  const month = t(`calendar.months.${monthKey}`);
  const year = d.format("YYYY");
  return `${day} ${month} ${year}`;
}

// ====================
// Show Date
// ====================
export function showDate(
  date: string | Date,
  dateFormat: DATE_FORMAT,
  splitWith?: string,
) {
  const d = dayjs(date);

  return `${d
    .format(dateFormat)
    .split("/")
    .join(splitWith ?? "-")}`;
}

// ====================
// Show Date with Hour
// ====================
export function showDateWithHour(
  date: string | Date,
  dateFormat: DATE_FORMAT,
  timeFormat: TIME_FORMAT,
  splitWith?: string,
) {
  const d = dayjs(date);

  const timePattern = timeFormat === TIME_FORMAT.H24 ? "HH:mm" : "hh:mm A";

  return `${d
    .format(dateFormat)
    .split("/")
    .join(splitWith ?? "-")} ${d.format(timePattern)}`;
}

export function formatTimeToSeconds(seconds: number) {
  const mins = Math.floor(seconds / 60);
  const secs = seconds % 60;
  return `${mins.toString().padStart(2, "0")}:${secs.toString().padStart(2, "0")}`;
}

// ====================
// Format Phone Number
// ====================
export function formatPhoneNumber(
  value: string,
  mask: string,
  countryCode?: string,
) {
  if (!value) return "";

  let formatted = "";
  let valueIndex = 0;

  for (let i = 0; i < mask.length && valueIndex < value.length; i++) {
    if (mask[i] === "0") {
      formatted += value[valueIndex];
      valueIndex++;
    } else {
      formatted += mask[i];
    }
  }

  if (countryCode) return `${countryCode} ${formatted}`;
  return formatted;
}

export function getRelativeTime(date: string | Date) {
  const now = dayjs();
  const d = dayjs(date);

  const diffInMinutes = now.diff(d, "minute");
  const diffInHours = now.diff(d, "hour");
  const diffInDays = now.diff(d, "day");
  const diffInWeeks = now.diff(d, "week");
  const diffInMonths = now.diff(d, "month");
  const diffInYears = now.diff(d, "year");

  // Less than 1 minute
  if (diffInMinutes < 1) {
    return t("calendar.relative.just_now");
  }

  // Less than 1 hour (e.g., 45m)
  if (diffInMinutes < 60) {
    return t("calendar.relative.m", { minute: diffInMinutes });
  }

  // Less than 24 hours (e.g., 3h)
  if (diffInHours < 24) {
    return t("calendar.relative.h", { hour: diffInHours });
  }

  // Less than 7 days (e.g., 5d)
  if (diffInDays < 7) {
    return t("calendar.relative.d", { day: diffInDays });
  }

  if (diffInWeeks < 4) {
    return t("calendar.relative.w", { week: diffInWeeks });
  }

  if (diffInMonths < 12) {
    return t("calendar.relative.mo", { month: diffInMonths });
  }

  return t("calendar.relative.y", { year: diffInYears });
}

```

## File: connectfy-client/src/common/utils/getDirtyValues.ts
```typescript
export function getChangedData<T extends Record<string, any>>(
  original: T,
  current: T,
): Partial<T> {
  const changes: Partial<T> = {};

  // Iterate over the keys of the current object
  Object.keys(current).forEach((key) => {
    const k = key as keyof T;
    // Strict equality check (adjust for deep comparison if you have nested objects)
    if (original[k] !== current[k]) {
      changes[k] = current[k];
    }
  });

  return changes;
}

```

## File: connectfy-client/src/common/helpers/history.ts
```typescript
import { createBrowserHistory } from "history";
export const history = createBrowserHistory();

```

## File: connectfy-client/src/common/helpers/security.events.ts
```typescript
import { SecurityEvent } from "../types/types";

const listeners: Record<SecurityEvent, Function[]> = {
  FORCE_LOGOUT: [],
  SESSION_EXPIRED: [],
};

export const securityEvents = {
  on(event: SecurityEvent, cb: Function) {
    listeners[event].push(cb);
  },
  emit(event: SecurityEvent) {
    listeners[event].forEach((cb) => cb());
  },
};

```

## File: connectfy-client/src/common/types/types.ts
```typescript
export type ApiErrorType = string | string[] | null | undefined;

export type SecurityEvent = "FORCE_LOGOUT" | "SESSION_EXPIRED";

export type AuthTokenManagerType = "access_token" | "authenticateToken" | "all";

```

## File: connectfy-client/src/common/api/axios.ts
```typescript
import {
  CustomAxiosRequestConfig,
  FailedRequest,
} from "@/common/interfaces/interfaces";
import { LANGUAGE, LOCAL_STORAGE_KEYS } from "@/common/enums/enums";
import axios, { AxiosError } from "axios";
import { API_ENDPOINTS } from "../constants/apiEndpoints";
import { checkDeviceId } from "../utils/checkValues";
import { securityEvents } from "../helpers/security.events";
import { baseUrl } from "../constants/constants";

export const BASE = `${baseUrl}/api/v1`;

export const api = axios.create({
  baseURL: BASE,
  withCredentials: true,
  timeout: 30_000,
});

export const refreshClient = axios.create({
  baseURL: BASE,
  withCredentials: true,
  timeout: 15_000,
});

let isRefreshing = false;
let failedQueue: FailedRequest[] = [];

const processQueue = (error: any = null, token: string | null = null) => {
  failedQueue.forEach((p) => {
    if (error) p.reject(error);
    else p.resolve(token!);
  });
  failedQueue = [];
};

const handleForceLogout = () => {
  localStorage.removeItem(LOCAL_STORAGE_KEYS.ACCESS_TOKEN);
  securityEvents.emit("FORCE_LOGOUT");
};

// --- REQUEST INTERCEPTOR ---
api.interceptors.request.use(
  (config: CustomAxiosRequestConfig) => {
    const _lang = localStorage.getItem(LOCAL_STORAGE_KEYS.LANG) || LANGUAGE.EN;
    const token = localStorage.getItem(LOCAL_STORAGE_KEYS.ACCESS_TOKEN);
    const isRefreshEndpoint = config.url?.includes(API_ENDPOINTS.AUTH.REFRESH);
    const deviceId = checkDeviceId();

    config.headers = config.headers || {};
    config.headers["x-device-id"] = deviceId;

    if (token && !isRefreshEndpoint) {
      config.headers = config.headers || {};
      config.headers.Authorization = `Bearer ${token}`;
    }

    // Attach _lang logic...
    if ((config.method || "get").toLowerCase() === "get") {
      config.params = { ...(config.params || {}), _lang };
    } else {
      if (config.data instanceof FormData) {
        config.data.append("_lang", _lang);
      } else if (typeof config.data === "string") {
        try {
          const parsed = JSON.parse(config.data);
          parsed._lang = _lang;
          config.data = JSON.stringify(parsed);
        } catch {
          config.data = `${config.data}&_lang=${_lang}`;
        }
      } else {
        config.data = { ...(config.data || {}), _lang };
      }
    }

    return config;
  },
  (error) => Promise.reject(error),
);

// --- RESPONSE INTERCEPTOR ---
api.interceptors.response.use(
  (res) => res,
  async (error: AxiosError) => {
    const originalRequest = error.config as CustomAxiosRequestConfig;
    if (!originalRequest) return Promise.reject(error);

    const errorData = error.response?.data as any;
    const status = errorData?.error?.statusCode || error.response?.status;

    // Check if backend explicitly instructed a navigation (force logout)
    // Note: Adjust the exact path based on how your BaseException serializes in NestJS
    const navigate =
      errorData?.navigate ||
      errorData?.error?.data?.navigate ||
      errorData?.data?.navigate;

    if (navigate) {
      handleForceLogout();
      return Promise.reject(error);
    }

    const isRefreshEndpoint = originalRequest.url?.includes(
      API_ENDPOINTS.AUTH.REFRESH,
    );

    // If 401, not a forced navigate, and not already retried
    if (status === 401 && !originalRequest._retry && !isRefreshEndpoint) {
      originalRequest._retry = true;

      if (isRefreshing) {
        return new Promise((resolve, reject) => {
          failedQueue.push({
            resolve: (token: string) => {
              originalRequest.headers = originalRequest.headers || {};
              originalRequest.headers.Authorization = `Bearer ${token}`;
              resolve(api.request(originalRequest));
            },
            reject,
          });
        });
      }

      isRefreshing = true;

      try {
        const refreshUrl = API_ENDPOINTS.AUTH.REFRESH.startsWith("http")
          ? API_ENDPOINTS.AUTH.REFRESH
          : `${BASE}${API_ENDPOINTS.AUTH.REFRESH}`;

        const _lang =
          localStorage.getItem(LOCAL_STORAGE_KEYS.LANG) || LANGUAGE.EN;
        const deviceId = checkDeviceId();

        const resp = await refreshClient.post(
          refreshUrl,
          { _lang },
          { withCredentials: true, headers: { "x-device-id": deviceId } },
        );

        const newAccessToken = (resp.data as any)?.access_token;
        if (!newAccessToken) throw new Error("No access token provided");

        localStorage.setItem(LOCAL_STORAGE_KEYS.ACCESS_TOKEN, newAccessToken);

        // Resolve queued requests
        processQueue(null, newAccessToken);

        // Retry original request
        originalRequest.headers = originalRequest.headers || {};
        originalRequest.headers.Authorization = `Bearer ${newAccessToken}`;
        return api.request(originalRequest);
      } catch (refreshErr: any) {
        // If the refresh request itself fails (or returns navigate: true)
        processQueue(refreshErr, null);
        handleForceLogout();
        return Promise.reject(refreshErr);
      } finally {
        isRefreshing = false;
      }
    }

    return Promise.reject(error);
  },
);

export default api;

```

## File: connectfy-client/src/common/api/axiosBaseQuery.ts
```typescript
import { BaseQueryFn } from "@reduxjs/toolkit/query";
import { api } from "./axios";

export type AxiosQueryArgs = {
  url: string;
  method: "GET" | "POST" | "PATCH" | "DELETE" | "PUT";
  data?: any;
  body?: any;
  params?: any;
};

export const baseQuery: BaseQueryFn<AxiosQueryArgs, unknown, unknown> = async ({
  url,
  method,
  data,
  body,
  params,
}) => {
  try {
    const res = await api({ url, method, data: body ?? data, params });
    return { data: res.data };
  } catch (err: any) {
    return { error: err.response?.data || { message: err.message } };
  }
};

```

## File: connectfy-client/src/common/interfaces/interfaces.ts
```typescript
import { InternalAxiosRequestConfig } from "axios";

export interface ICountry {
  key: string;
  name: string;
  flag: string;
  code: string;
  numberLength: number;
  format: string;
  totalLength: number;
}

export interface IPagination {
  // pageSize?: number | undefined;
  // current?: number;
  totalCount: number;
  // totalPages: number;
  // next?: number | null;
  // prev?: number | null;
}

export interface FailedRequest {
  resolve: (token: string) => void;
  reject: (error: any) => void;
}

export interface CustomAxiosRequestConfig extends InternalAxiosRequestConfig {
  _retry?: boolean;
}

export interface IUpdateResponse {
  success: boolean;
  [key: string]: any;
}

export interface IFindAllResponse<T> {
  data: T[];
  totalCount?: number;
  limit?: number;
  currentPage?: number;
  totalPages?: number;
}

export interface IRemoveResponse {
  success: boolean;
  [key: string]: any;
}

export interface IRemoveAllResponse {
  deletedIds: string[];
  notDeleted: string[];
  deletedCount: number;
  [key: string]: any;
}

export interface IRemoveData {
  _id: string;
  [key: string]: any;
}

export interface IRemoveAllData {
  _ids: string[];
  [key: string]: any;
}

```

