# Project Documentation

This document provides comprehensive documentation for the Customer Webapp, covering API patterns, form validation, custom hooks, state management, component architecture, route protection, and infinite scrolling with virtualization.

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Project Structure](#2-project-structure)
3. [Component Architecture](#3-component-architecture)
4. [Protect the Route based on User Role](#4-protect-the-route-based-on-user-role)
5. [Form Validation with Zod and React Hook Form](#5-form-validation-with-zod-and-react-hook-form)
6. [Jobseeker Schemas](#6-jobseeker-schemas)
7. [API Calls with TanStack Query](#7-api-calls-with-tanstack-query)
8. [Zustand Store Management](#8-zustand-store-management)
9. [Custom Hooks](#9-custom-hooks)

---

## 1. Project Overview

This is a Next.js 15 application built with React 19, implementing a job marketplace platform with two main user roles:

- **Job Seekers**: Browse jobs, apply, manage profile
- **Employers**: Post jobs, manage applications, browse candidates

### Key Technologies

- **Framework**: Next.js 15 (App Router)
- **State Management**: Zustand
- **Data Fetching**: TanStack Query (React Query)
- **Form Management**: React Hook Form
- **Validation**: Zod
- **HTTP Client**: Axios
- **Styling**: Tailwind CSS

---

## 2. Project Structure

```
customer-webapp/
├── src/
│   ├── apis/                    # API hooks (TanStack Query)
│   │   ├── auth.ts              # Authentication APIs
│   │   ├── jobseeker.ts         # Jobseeker CRUD operations
│   │   ├── employer.ts          # Employer CRUD operations
│   │   ├── jobs.ts              # Job listing/search APIs
│   │   └── common.ts            # Common/shared APIs
│   │
│   ├── app/                     # Next.js App Router
│   │   ├── layout.tsx           # Root layout
│   │   ├── page.tsx             # Home page
│   │   ├── sign-in/             # Sign in page
│   │   ├── sign-up/             # Sign up page
│   │   ├── profile/             # Profile pages
│   │   ├── jobs/                # Job listing pages
│   │   └── employer/            # Employer dashboard pages
│   │
│   ├── components/              # React components
│   │   ├── common/              # Reusable UI components
│   │   ├── form/                # Form input components
│   │   ├── dashboard/           # Dashboard components
│   │   └── profile/             # Profile components
│   │
│   ├── sections/                # Page sections
│   │   ├── auth-section/        # Auth forms
│   │   ├── dashboard/           # Dashboard sections
│   │   ├── profile/             # Profile sections
│   │   └── jobs/                # Job sections
│   │
│   ├── store/                   # Zustand stores
│   │   ├── auth-store.ts        # Authentication state
│   │   ├── layout-store.ts      # Layout/UI state
│   │   └── notification-store.ts # Notification state
│   │
│   ├── hooks/                   # Custom React hooks
│   │   ├── use-pagination.ts    # Pagination logic
│   │   ├── use-location-data.ts # Location data
│   │   └── use-sse-notification.ts # SSE notifications
│   │
│   ├── schema/                  # Zod validation schemas
│   │   ├── registration-schema.ts # Registration forms
│   │   └── employer-schema.ts   # Employer forms
│   │
│   ├── utils/                   # Utility functions
│   │   ├── axios.ts             # Axios instance & endpoints
│   │   ├── helper.ts            # Helper functions
│   │   ├── interface.ts         # TypeScript interfaces
│   │   └── session-storage.ts   # Session storage helpers
│   │
│   ├── layouts/                 # Layout components
│   │   ├── main-layout.tsx      # Main app layout
│   │   ├── dashboard-layout.tsx # Dashboard layout
│   │   ├── navbar.tsx           # Navigation bar
│   │   └── sidebar.tsx          # Sidebar component
│   │
│   └── auth/                    # Auth guards
│       ├── jobseeker-auth-guard.tsx
│       ├── employer-auth-guard.tsx
│       └── guest-guard.tsx
│
├── public/                      # Static assets
├── package.json                 # Dependencies
├── tsconfig.json                # TypeScript config
├── tailwind.config.ts           # Tailwind CSS config
└── next.config.ts               # Next.js config
```

---

## 3. Component Architecture

### Component Organization

```
src/
├── components/
│   ├── common/          # Reusable UI components
│   ├── form/            # Form input components
│   ├── dashboard/        # Dashboard-specific components
│   ├── company/         # Company-related components
│   └── profile/          # Profile-related components
├── sections/            # Page sections and complex components
├── layouts/             # Layout components
└── app/                 # Next.js App Router pages
```

### Common Components

Located in `src/components/common/`:

- `pagination.tsx` - Pagination controls
- `job-widget.tsx` - Job listing card
- `company-widget.tsx` - Company card
- `modal.tsx` - Modal dialog
- `dropdown.tsx` - Dropdown menu
- `stepper.tsx` - Multi-step form stepper
- `logo.tsx` - Logo component
- `avatar-progress.tsx` - Avatar with progress indicator

### Form Components

All form components in `src/components/form/` use React Hook Form's `Controller` and integrate with Zod validation.

**Example: TextArea Input**

```typescript
// src/components/form/textarea-input.tsx
export default function RHFTextAreaField({
  name,
  label,
  required,
  validationType = "characters", // "characters" | "words"
  maxCharacters = 500,
  maxWords = 500,
  ...other
}: Props) {
  const { control } = useFormContext();

  return (
    <Controller
      name={name}
      control={control}
      render={({ field, fieldState: { error } }) => (
        <div className="flex flex-col w-full">
          <textarea {...field} {...other} />
          {/* Word/character count display */}
        </div>
      )}
    />
  );
}
```

### Section Components

Complex page sections in `src/sections/`:

- `auth-section/` - Authentication forms and flows
- `dashboard/` - Dashboard sections
- `profile/` - Profile management sections
- `jobs/` - Job listing and details sections
- `registration/` - Registration flow sections

---

## 4. Protect the Route based on User Role

We use **Auth Guards** to protect routes based on user authentication status and role. There are three types of guards:

### Jobseeker Auth Guard

Protects routes that require jobseeker authentication:

```typescript
// src/auth/jobseeker-auth-guard.tsx
"use client";

import { useEffect, useCallback, useState } from "react";
import { useAuthIsAuthenticated, useAuthRole } from "@/store/auth-store";
import { useRouter } from "next/navigation";

type Props = {
  children: React.ReactNode;
};

export default function JobSeekerAuthGuard({ children }: Props) {
  const router = useRouter();
  const [checked, setChecked] = useState(false);

  const isAuthenticated = useAuthIsAuthenticated();
  const role = useAuthRole();

  // Check user authentication, for unauthorized redirect to Welcome page
  const check = useCallback(() => {
    if (!isAuthenticated) {
      router.replace("/");
    } else {
      if (role === "job_seeker") {
        setChecked(true);
      } else if (role === "employer") {
        router.replace("/employer/dashboard");
      } else {
        router.replace("/");
      }
    }
  }, [isAuthenticated, role, router]);

  useEffect(() => {
    check();
  }, [isAuthenticated, role, check]);

  if (!checked) {
    return null;
  }
  return <>{children}</>;
}
```

**Usage in Layout:**

```typescript
// src/app/profile/layout.tsx
import JobSeekerAuthGuard from "@/auth/jobseeker-auth-guard";

export default function ProfileLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return <JobSeekerAuthGuard>{children}</JobSeekerAuthGuard>;
}
```

### Employer Auth Guard

Protects routes that require employer authentication:

```typescript
// src/auth/employer-auth-guard.tsx
"use client";

import { useEffect, useCallback, useState } from "react";
import { useAuthIsAuthenticated, useAuthRole } from "@/store/auth-store";
import { useRouter } from "next/navigation";

type Props = {
  children: React.ReactNode;
};

export default function EmployerAuthGuard({ children }: Props) {
  const router = useRouter();
  const [checked, setChecked] = useState(false);

  const isAuthenticated = useAuthIsAuthenticated();
  const role = useAuthRole();

  // Check user authentication, for unauthorized redirect to Welcome page
  const check = useCallback(() => {
    if (!isAuthenticated) {
      router.replace("/");
    } else {
      if (role === "employer") {
        setChecked(true);
      } else if (role === "job_seeker") {
        router.replace("/home");
      } else {
        router.replace("/");
      }
    }
  }, [isAuthenticated, role, router]);

  useEffect(() => {
    check();
  }, [isAuthenticated, role, check]);

  if (!checked) {
    return null;
  }
  return <>{children}</>;
}
```

**Usage in Layout:**

```typescript
// src/app/employer/dashboard/layout.tsx
import EmployerAuthGuard from "@/auth/employer-auth-guard";

export default function EmployerDashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return <EmployerAuthGuard>{children}</EmployerAuthGuard>;
}
```

### Guest Guard

Protects routes that should only be accessible to unauthenticated users (e.g., sign-in, sign-up):

```typescript
// src/auth/guest-guard.tsx
"use client";

import { useCallback, useEffect, useState } from "react";
import { useAuthIsAuthenticated, useAuthRole } from "@/store/auth-store";
import { useRouter } from "next/navigation";

type Props = {
  children: React.ReactNode;
};

export default function GuestGuard({ children }: Props) {
  const router = useRouter();
  const [checked, setChecked] = useState(false);
  const isAuthenticated = useAuthIsAuthenticated();
  const role = useAuthRole();

  const check = useCallback(() => {
    if (isAuthenticated && role === "employer") {
      router.replace("/employer/dashboard");
    } else if (isAuthenticated && role === "job_seeker") {
      router.replace("/home");
    } else {
      setChecked(true);
    }
  }, [isAuthenticated, role, router]);

  useEffect(() => {
    check();
  }, [check]);

  if (!checked) {
    return null;
  }

  return <>{children}</>;
}
```

**Usage in Layout:**

```typescript
// src/app/sign-in/layout.tsx
import GuestGuard from "@/auth/guest-guard";

export default function SignInLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return <GuestGuard>{children}</GuestGuard>;
}
```

### Guard Behavior Summary

| Guard Type           | Allows                   | Redirects To                                                |
| -------------------- | ------------------------ | ----------------------------------------------------------- |
| `JobSeekerAuthGuard` | Authenticated job_seeker | `/` if not authenticated, `/employer/dashboard` if employer |
| `EmployerAuthGuard`  | Authenticated employer   | `/` if not authenticated, `/home` if job_seeker             |
| `GuestGuard`         | Unauthenticated users    | `/employer/dashboard` if employer, `/home` if job_seeker    |

---

## 5. Form Validation with Zod and React Hook Form

We use **TanStack Query** (React Query) for all API operations, providing caching, background updates, and error handling out of the box.

### Setup

The React Query provider is configured in `src/app/react-query-provider.tsx`:

```typescript
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000, // Data is fresh for 60 seconds
      },
    },
  });
}
```

### CRUD Operations Pattern

All API calls follow a consistent pattern organized by resource type in the `src/apis/` directory.

#### 1. **CREATE Operation (POST)**

```typescript
// src/apis/employer.ts
export type JobPostingData = {
  title: string;
  description: string;
  category: string;
  job_type: string;
  openings: number;
  required_experience: string;
  skills: string[];
  expires_at: string | null;
  salary_min: number | null;
  salary_max: number | null;
  salary_currency: string | null;
  city: string | null;
  state: string | null;
  country: string | null;
  is_active: boolean;
  hide_salary: boolean;
  is_remote: boolean;
  education_qualification: string | null;
};

export const useCreateJobPosting = () => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async (data: JobPostingData) => {
      const res = await axiosInstance.post<NullableResponseBase>(
        endPoints.employer.createJobPosting,
        data
      );
      return res.data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["jobposts"] });
    },
  });
};
```

**Example from `auth.ts` - Jobseeker Registration with File Upload:**

```typescript
// src/apis/auth.ts
export const useRegisterJobseeker = () => {
  return useMutation({
    mutationFn: async (data: FormData) => {
      const response = await axiosInstance.post<null>(
        endPoints.auth.jobSeekerRegister,
        data,
        {
          headers: {
            "Content-Type": "multipart/form-data",
          },
        }
      );
      return response.data;
    },
    onSuccess: (data) => {
      console.log("register jobseeker successfully:", data);
      return data;
    },
    onError: (error) => {
      console.log("register jobseeker failed:", error);
      return error;
    },
  });
};
```

#### 2. **READ Operation (GET)**

**Example from `employer.ts` - Fetching Employer Profile:**

```typescript
// src/apis/employer.ts
type EmployerMeResponse = {
  status: number;
  message: string;
  data: {
    id: string;
    company_logo: string;
    company_name: string;
    company_website: string;
    company_size: string;
    about_company?: string;
    industry?: string;
    phone_number?: string;
    country_code?: string;
    email?: string;
    name: string;
    email_verified?: boolean;
    phone_verified?: boolean;
    city?: string;
    state?: string;
    country?: string;
    user_id: string;
    created_at: string;
    updated_at?: string;
  };
  error: string;
};

export const useGetEmployeMe = ({
  isAuth = false,
  role,
}: {
  isAuth?: boolean;
  role: "employer" | "job_seeker" | null;
}) => {
  return useQuery({
    queryKey: ["employerMe"],
    queryFn: async () => {
      const res = await axiosInstance.get<EmployerMeResponse>(
        endPoints.employer.me
      );
      return res.data.data;
    },
    enabled: isAuth && role === "employer", // Conditional fetching
  });
};
```

**Example from `jobs.ts` - Fetching Job Details:**

```typescript
// src/apis/jobs.ts
type JobDetailsResponse = {
  status: number;
  message: string;
  data: JobDetalsResult;
  error: string;
};

export const useGetJobDetails = (params: JobDetailsParams) => {
  return useQuery({
    queryKey: ["jobDetails", params],
    queryFn: async () => {
      const res = await axiosInstance.get<JobDetailsResponse>(
        `${endPoints.jobseeker.jobs.jobDetails.replace("{id}", params.id)}`
      );
      return res.data.data;
    },
  });
};
```

**Example from `employer.ts` - Fetching Job Postings with Pagination:**

```typescript
// src/apis/employer.ts
type JobPostingParams = {
  page: number;
  size: number;
  search?: string;
  is_active?: boolean;
};

export const useGetJobposts = (params: JobPostingParams) => {
  return useQuery({
    queryKey: ["jobposts", params],
    queryFn: async () => {
      const res = await axiosInstance.get<JobPostingsResponse>(
        endPoints.employer.createJobPosting,
        {
          params,
        }
      );
      return res.data.data;
    },
  });
};
```

#### 3. **UPDATE Operation (PUT)**

**Example from `employer.ts` - Updating Employer Profile with File Upload:**

```typescript
// src/apis/employer.ts
export const useUpdateEmployerMe = () => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async (data: FormData) => {
      const res = await axiosInstance.put<NullableResponseBase>(
        endPoints.employer.me,
        data,
        {
          headers: {
            "Content-Type": "multipart/form-data",
          },
        }
      );
      return res.data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["employerMe"] });
    },
  });
};
```

**Example from `employer.ts` - Closing Job Posting:**

```typescript
// src/apis/employer.ts
export const useCloseJobPosting = () => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async (id: string) => {
      const res = await axiosInstance.put<NullableResponseBase>(
        endPoints.employer.jobPostingDetails.replace("{id}", id)
      );
      return res.data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["jobPostingDetails"] });
      queryClient.invalidateQueries({ queryKey: ["jobposts"] });
    },
  });
};
```

#### 4. **DELETE Operation (DELETE)**

**Example from `common.ts` - Deleting Notification:**

```typescript
// src/apis/common.ts
type NotificationParams = {
  id: string;
};

export const useDeleteNotification = () => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async (params: NotificationParams) => {
      const res = await axiosInstance.delete<NullableResponseBase>(
        endPoints.notification.notificationDelete.replace("{id}", params.id)
      );
      return res.data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["notificationMe"] });
      queryClient.invalidateQueries({ queryKey: ["notificationStats"] });
    },
  });
};
```

### Authentication Examples from `auth.ts`

**Login with OTP Verification:**

```typescript
// src/apis/auth.ts
type LoginParams = {
  email: string;
  password: string;
  role: string;
};

export const useJobSeekerLogin = () => {
  return useMutation({
    mutationFn: async (params: LoginParams) => {
      const response = await axiosInstance.post<NullableResponseBase>(
        endPoints.auth.login,
        params
      );
      return response.data;
    },
  });
};

export const useJobSeekerLoginVerify = () => {
  const queryClient = useQueryClient();
  const { setAuthUserDetails, setAuthToken } = useAuthActions();

  return useMutation({
    mutationFn: async (params: EmailVerifyParams) => {
      const response = await axiosInstance.post<LoginResponse>(
        endPoints.auth.loginVerify,
        params
      );
      return response.data;
    },
    onSuccess: async (_loginData) => {
      setAuthToken(_loginData);
      axiosInstance.defaults.headers.common.Authorization = `Bearer ${_loginData.access_token}`;
    },
  });
};
```

**Social Authentication (Google):**

```typescript
// src/apis/auth.ts
export const useGetGoogleAuthorizationUrl = () => {
  return useMutation({
    mutationFn: async (params: socialAuthParams) => {
      const response = await axiosInstance.get<SocialAuthorizationUrlResponse>(
        `${endPoints.auth.social.google}?role=${params.role}`
      );
      return response.data;
    },
  });
};

export const useJobSeekerGoogleLogin = () => {
  const { setAuthToken } = useAuthActions();

  return useMutation({
    mutationFn: async (params: SocialLoginParams) => {
      const response = await axiosInstance.post<LoginResponse>(
        `${endPoints.auth.social.googleCallback}?code=${params.code}&role=${params.role}`
      );
      return response.data;
    },
    onSuccess: async (_loginData) => {
      setAuthToken(_loginData);
      axiosInstance.defaults.headers.common.Authorization = `Bearer ${_loginData.access_token}`;
    },
  });
};
```

**Password Reset:**

```typescript
// src/apis/auth.ts
type PasswordResetParams = {
  email: string;
  role: string;
};

export const useAuthPasswordReset = () => {
  return useMutation({
    mutationFn: async (params: PasswordResetParams) => {
      const response = await axiosInstance.post<NullableResponseBase>(
        `${endPoints.auth.passwordResetLink.replace(
          "{email}",
          params.email
        )}?role=${params.role}`
      );
      return response.data;
    },
  });
};

type PasswordResetTokenParams = {
  token: string;
  new_password: string;
};

export const useAuthPasswordResetToken = () => {
  return useMutation({
    mutationFn: async (params: PasswordResetTokenParams) => {
      const response = await axiosInstance.post<NullableResponseBase>(
        endPoints.auth.passwordReset.replace("{token}", params.token),
        {
          new_password: params.new_password,
        }
      );
      return response.data;
    },
  });
};
```

### TanStack Infinite Query with Virtualization

We use **TanStack Infinite Query** combined with **TanStack Virtual** for efficient infinite scrolling with virtualization.

#### Infinite Query Setup

**Example from `jobs.ts` - Infinite Job Search:**

```typescript
// src/apis/jobs.ts
export const useInfiniteJobSearch = (
  params: Omit<JobListingParams, "page" | "limit">
) => {
  return useInfiniteQuery({
    queryKey: ["infiniteJobSearch", params],
    initialPageParam: 1,
    queryFn: async ({ pageParam }) => {
      const queryParams = new URLSearchParams();

      // Add all non-empty parameters to query string
      Object.entries(params).forEach(([key, value]) => {
        if (value && value !== "") {
          queryParams.append(key, value.toString());
        }
      });

      // Add pagination params
      queryParams.append("page", pageParam.toString());
      queryParams.append("size", "10");

      const res = await axiosInstance.get<JobListingResponse>(
        `${endPoints.jobseeker.jobs.searchJobs}?${queryParams.toString()}`
      );
      return {
        results: res.data.data.results,
        pagination: res.data.data.pagination,
        total: res.data.data.pagination.total,
      };
    },
    getNextPageParam: (lastPage) => {
      // Check if there's a next page
      if (lastPage.pagination.has_next) {
        return lastPage.pagination.page + 1;
      }
      return undefined;
    },
    select: (data) => {
      return {
        jobs: data.pages.flatMap((page) => page.results),
        total: data.pages[0]?.total || 0,
        pagination: data.pages[data.pages.length - 1]?.pagination,
      };
    },
    refetchOnWindowFocus: false,
    refetchOnMount: false,
    refetchOnReconnect: false,
    staleTime: 5 * 60 * 1000, // Consider data fresh for 5 minutes
  });
};
```

**Example from `common.ts` - Infinite Notifications:**

```typescript
// src/apis/common.ts
export const useInfiniteNotifiationQuery = (isAuth: boolean) => {
  return useInfiniteQuery({
    enabled: isAuth,
    queryKey: ["notificationMe"],
    initialPageParam: 1,
    queryFn: async ({ pageParam }) => {
      const res = await axiosInstance.get<NotificationMeResponse>(
        endPoints.notification.notifications,
        {
          params: {
            page: pageParam,
            size: 5,
          },
        }
      );

      return {
        notifications: res.data.data.notifications,
        total: res.data.data.total,
        unread_count: res.data.data.unread_count,
        page: res.data.data.page,
        size: res.data.data.size,
      };
    },
    select: (data) => {
      return {
        notifications: data.pages.flatMap((page) => page.notifications),
        total: data.pages[0]?.total,
        unread_count: data.pages[0]?.unread_count,
        page: data.pages[0]?.page,
        size: data.pages[0]?.size,
      };
    },
    getNextPageParam: (lastPage, allPages) => {
      const totalFetched = allPages.length * lastPage?.size;
      const nextPage = allPages.length + 1;
      return totalFetched < (lastPage?.total || 0) ? nextPage : undefined;
    },
  });
};
```

#### Virtualization with TanStack Virtual

**Example Component - Virtualized Job List:**

```typescript
// src/sections/jobs/job-details-section.tsx
import { useVirtualizer } from "@tanstack/react-virtual";
import { useInfiniteJobSearch, JobResult } from "@/apis/jobs";

interface JobsVirtualListProps {
  jobPostId: string;
  jobs: JobResult[];
  fetchNextPage: (options?: FetchNextPageOptions) => void;
  hasNextPage: boolean;
  isLoading: boolean;
  isFetching: boolean;
  isFetchingNextPage: boolean;
  maxHeight?: number;
  parentRef: RefObject<HTMLDivElement>;
  queryParams: Record<string, string>;
}

function JobsVirtualList({
  jobPostId,
  jobs,
  fetchNextPage,
  hasNextPage,
  isLoading,
  isFetching,
  isFetchingNextPage,
  maxHeight,
  parentRef,
  queryParams,
}: JobsVirtualListProps) {
  const jobList = useMemo(() => jobs.filter((job) => Boolean(job)), [jobs]);
  const count = hasNextPage ? jobList.length + 1 : jobList.length;
  const lastFetchedIndexRef = useRef<number>(-1);

  // Initialize virtualizer
  const virtualizer = useVirtualizer({
    count,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 220, // Estimated height of each item
    overscan: 10, // Render 10 extra items outside viewport
    measureElement: (el) => el?.getBoundingClientRect().height || 0,
  });

  const virtualItems = virtualizer.getVirtualItems();

  // Fetch next page when scrolling near the end
  useEffect(() => {
    if (virtualItems.length === 0 || jobList.length === 0) {
      return;
    }

    const lastItem = virtualItems[virtualItems.length - 1];

    // Fetch if:
    // 1. Last item is near the end (within last 2 items)
    // 2. There's a next page available
    // 3. We're not already fetching
    // 4. We haven't already fetched for this index
    if (
      lastItem &&
      lastItem.index >= jobList.length - 2 &&
      hasNextPage &&
      !isFetchingNextPage &&
      !isFetching &&
      lastItem.index !== lastFetchedIndexRef.current
    ) {
      lastFetchedIndexRef.current = lastItem.index;
      fetchNextPage();
    }
  }, [
    virtualItems,
    jobList.length,
    hasNextPage,
    fetchNextPage,
    isFetchingNextPage,
    isFetching,
  ]);

  // Reset ref when jobList length changes
  useEffect(() => {
    lastFetchedIndexRef.current = -1;
  }, [jobList.length]);

  if (isLoading && jobList.length === 0) {
    return <div>Loading...</div>;
  }

  if (!isLoading && !isFetching && jobList.length === 0) {
    return <div>No jobs found</div>;
  }

  return (
    <div
      ref={parentRef}
      className="overflow-y-auto scrollbar-hide"
      style={{ height: maxHeight ? `${maxHeight}px` : "70vh" }}
    >
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          width: "100%",
          position: "relative",
        }}
      >
        {virtualItems.map((virtualItem) => {
          const job = jobList[virtualItem.index];
          const isLoaderRow = virtualItem.index >= jobList.length;

          // Show loading indicator at the end
          if (isLoaderRow) {
            return (
              <div
                key={virtualItem.key}
                data-index={virtualItem.index}
                ref={virtualizer.measureElement}
                style={{
                  position: "absolute",
                  top: 0,
                  left: 0,
                  width: "100%",
                  transform: `translateY(${virtualItem.start}px)`,
                  paddingBottom: "16px",
                }}
              >
                {hasNextPage && (
                  <div className="flex items-center justify-center py-4">
                    Loading more...
                  </div>
                )}
              </div>
            );
          }

          if (!job) {
            return null;
          }

          // Render actual job item
          return (
            <div
              key={virtualItem.key}
              data-index={virtualItem.index}
              ref={virtualizer.measureElement}
              style={{
                position: "absolute",
                top: 0,
                left: 0,
                width: "100%",
                transform: `translateY(${virtualItem.start}px)`,
              }}
            >
              <JobCard
                jobId={job.id}
                companyName={job.company_name}
                companyLogo={job.logo}
                jobTitle={job.title}
                location={job.location}
                workType={job.job_type}
                postedTime={getTimeAgo(job.created_at)}
                experience={job.required_experience}
                hideSalary={job.hide_salary}
                jobPostingId={jobPostId}
                isApplied={job.is_applied}
              />
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

**Usage in Parent Component:**

```typescript
export default function JobDetailsSection({
  jobPostId,
}: JobDetailsSectionProps) {
  const searchParams = useSearchParams();
  const parentRef = useRef<HTMLDivElement>(null);

  // Prepare query params
  const queryParams = useMemo(() => {
    return {
      q: searchParams.get("q") || undefined,
      location: searchParams.get("location") || undefined,
      job_type: searchParams.get("job_type") || undefined,
      // ... other params
    };
  }, [searchParams]);

  // Use infinite query
  const {
    data: jobSearchData,
    fetchNextPage,
    hasNextPage,
    isFetching,
    isFetchingNextPage,
    isLoading,
  } = useInfiniteJobSearch(queryParams);

  const jobs = jobSearchData?.jobs || [];

  return (
    <JobsVirtualList
      jobPostId={jobPostId}
      jobs={jobs}
      fetchNextPage={fetchNextPage}
      hasNextPage={hasNextPage || false}
      isLoading={isLoading}
      isFetching={isFetching}
      isFetchingNextPage={isFetchingNextPage}
      parentRef={parentRef}
      queryParams={queryParams}
    />
  );
}
```

### Query Invalidation Pattern

Here's a complete example showing all CRUD operations for language management:

```typescript
// src/apis/jobseeker.ts

// CREATE
export const useCreateLanguage = () => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async (data: LanguageParams) => {
      const res = await axiosInstance.post<NullableResponseBase>(
        endPoints.jobseeker.languages,
        data
      );
      return res.data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["jobseekerMe"] });
    },
  });
};

// READ (already included in useGetJobseekerMe)

// UPDATE
export const useUpdateLanguage = () => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async (data: UpdateLanguageParams) => {
      const { id, ...rest } = data;
      const res = await axiosInstance.put<NullableResponseBase>(
        endPoints.jobseeker.languagesDetails.replace("{id}", id),
        rest
      );
      return res.data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["jobseekerMe"] });
    },
  });
};

// DELETE
export const useDeleteLanguage = () => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async (id: string) => {
      const res = await axiosInstance.delete<NullableResponseBase>(
        endPoints.jobseeker.languagesDetails.replace("{id}", id)
      );
      return res.data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["jobseekerMe"] });
    },
  });
};
```

### Query Invalidation Pattern

After mutations, we invalidate related queries to ensure UI stays in sync:

```typescript
onSuccess: () => {
  queryClient.invalidateQueries({ queryKey: ["jobseekerMe"] });
  // Or invalidate multiple related queries
  queryClient.invalidateQueries({ queryKey: ["jobposts"] });
  queryClient.invalidateQueries({ queryKey: ["notificationMe"] });
};
```

### File Upload Pattern

For file uploads, use `FormData`:

```typescript
export const useUpdateJobseekerMe = () => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async (data: FormData) => {
      const res = await axiosInstance.put<NullableResponseBase>(
        endPoints.jobseeker.profile,
        data,
        {
          headers: {
            "Content-Type": "multipart/form-data",
          },
        }
      );
      return res.data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["jobseekerMe"] });
    },
  });
};
```

---

## 6. Jobseeker Schemas

All jobseeker-related validation schemas are defined in `src/schema/registration-schema.ts`. Here are the key schemas:

### Password Validation Schema

```typescript
export const passwordValidation = z
  .string()
  .min(8, "Password must be at least 8 characters long")
  .refine(
    (password) => /[A-Z]/.test(password),
    "Password must contain at least one uppercase letter"
  )
  .refine(
    (password) => /[a-z]/.test(password),
    "Password must contain at least one lowercase letter"
  )
  .refine(
    (password) => /[0-9]/.test(password),
    "Password must contain at least one number"
  )
  .refine(
    (password) => /[^A-Za-z0-9]/.test(password),
    "Password must contain at least one special character"
  );
```

### Work Experience Schema

```typescript
export const workExperienceSchema = z
  .object({
    job_title: z.string().min(1, "Job title is required"),
    company_name: z.string().min(1, "Company name is required"),
    is_current_company: z.boolean(),
    joining_date: z.object({
      year: z.string().min(1, "Joining year is required"),
      month: z.string().min(1, "Joining month is required"),
    }),
    end_date: z
      .object({
        year: z.string().min(1, "End year is required"),
        month: z.string().min(1, "End month is required"),
      })
      .optional(),
  })
  .refine(
    (data) => {
      // If not current company, end_date is required
      if (!data.is_current_company && !data.end_date) {
        return false;
      }
      return true;
    },
    {
      message: "End date is required for previous companies",
      path: ["end_date"],
    }
  )
  .refine(
    (data) => {
      // If end_date exists, validate that joining_date is before end_date
      if (data.end_date) {
        const joiningYear = parseInt(data.joining_date.year, 10);
        const joiningMonth = parseInt(data.joining_date.month, 10);
        const endYear = parseInt(data.end_date.year, 10);
        const endMonth = parseInt(data.end_date.month, 10);

        if (
          isNaN(joiningYear) ||
          isNaN(joiningMonth) ||
          isNaN(endYear) ||
          isNaN(endMonth)
        ) {
          return true; // Let other validations handle invalid inputs
        }

        if (joiningYear > endYear) {
          return false;
        }
        if (joiningYear === endYear && joiningMonth >= endMonth) {
          return false;
        }
      }
      return true;
    },
    {
      message: "Start date must be before end date",
      path: ["end_date"],
    }
  );
```

### Education Schema

```typescript
export const educationSchema = z
  .object({
    degree: z.string().min(1, "Degree is required"),
    university: z.string().min(1, "University is required"),
    start_year: z.string().min(1, "Start year is required"),
    end_year: z.string().min(1, "End year is required"),
  })
  .refine(
    (data) => {
      const startYear = parseInt(data.start_year, 10);
      const endYear = parseInt(data.end_year, 10);
      if (isNaN(startYear) || isNaN(endYear)) {
        return true;
      }
      return startYear <= endYear;
    },
    {
      message: "Start year must be before or equal to end year",
      path: ["start_year"],
    }
  );
```

### Certification Schema

```typescript
export const certificationSchema = z
  .object({
    certification_name: z.string().min(1, "Certification name is required"),
    issuing_organization: z.string().min(1, "Organization is required"),
    issue_date: z.date().nullable().optional(),
    expiry_date: z.date().nullable().optional(),
    description: z.string().optional(),
  })
  .refine(
    (data) => {
      if (data.expiry_date) {
        return data.issue_date && data.issue_date < data.expiry_date;
      }
      return true;
    },
    {
      message: "Issue date must be before expiry date",
      path: ["expiry_date"],
    }
  );
```

### Language Schema

```typescript
export const languageSchema = z.object({
  language_name: z.string().min(1, "Language name is required"),
  proficiency: z.enum(
    ["beginner", "intermediate", "proficient", "advanced", "native"],
    {
      message:
        "Proficiency must be beginner, intermediate, proficient, advanced or native",
    }
  ),
});
```

### Complete Jobseeker Registration Schema

```typescript
export const jobseekerRegistrationSchema = z
  .object({
    first_name: z
      .string()
      .min(1, "First name is required")
      .regex(/^[A-Za-z\s]+$/, "First name must contain only alphabets"),
    last_name: z
      .string()
      .min(1, "Last name is required")
      .regex(/^[A-Za-z\s]+$/, "Last name must contain only alphabets"),
    email: z
      .string()
      .min(1, "Email is required")
      .email("Invalid email address"),
    phone_number: z
      .string()
      .min(1, "Mobile number is required")
      .refine(
        (value) => {
          if (!value) return false;
          try {
            return isValidPhoneNumber(value, "TZ");
          } catch {
            return false;
          }
        },
        {
          message: "Please enter a valid mobile number",
        }
      ),
    password: passwordValidation,
    confirmPassword: z.string().min(1, "Confirm password is required"),
    gender: z.enum(["male", "female", "other"], {
      message: "Gender must be male, female or other",
    }),
    date_of_birth: z
      .object({
        date: z.string().min(1, "Birth date is required"),
        month: z.string().min(1, "Birth month is required"),
        year: z.string().min(1, "Birth year is required"),
      })
      .refine((data) => isValidDateForMonth(data.date, data.month, data.year), {
        message: "Invalid date for the selected month and year",
      })
      .refine((data) => isNotFutureDate(data.date, data.month, data.year), {
        message: "Date of birth cannot be in the future",
      })
      .refine((data) => isAtLeast18YearsOld(data.date, data.month, data.year), {
        message: "You must be at least 18 years old to register",
      }),
    country: lableValueSchema
      .nullable()
      .refine((data) => data !== null && data !== undefined, {
        message: "Country is required",
      }),
    state: lableValueSchema.optional(),
    city: lableValueSchema.optional(),
    profile_photo: z
      .any()
      .optional()
      .refine(
        (file) => !file || file?.size <= MAX_PROFILE_PHOTO_SIZE,
        `Max file size is 2MB.`
      )
      .refine(
        (file) => !file || ACCEPTED_IMAGE_TYPES.includes(file?.type),
        "Only .jpg, .jpeg, .png, and .webp formats are supported."
      ),
    resume: z
      .any()
      .optional()
      .refine(
        (file) => !file || file?.size <= MAX_RESUME_SIZE,
        `Max file size is 10MB.`
      )
      .refine(
        (file) => !file || ACCEPTED_RESUME_TYPES.includes(file?.type),
        "Only .pdf, .doc and .docx formats are supported."
      ),
    languages: z.array(
      z.object({
        language_name: z.string(),
        proficiency: z.string(),
      })
    ),
    tech_skills: z
      .array(z.string())
      .min(1, "At least one tech skill is required"),
    soft_skills: z
      .array(z.string())
      .min(1, "At least one soft skill is required"),
    work_experience: z.array(workExperienceSchema),
    education: z.array(educationSchema),
    certifications: z.array(certificationSchema),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: "Passwords don't match",
    path: ["confirmPassword"],
  })
  .superRefine((data, ctx) => {
    // Check for duplicate language names (case-insensitive)
    const languageNames = data.languages.map((lang, index) => ({
      name: lang.language_name?.toLowerCase().trim(),
      index,
    }));

    const seen = new Map<string, number[]>();

    languageNames.forEach(({ name, index }) => {
      if (name) {
        if (!seen.has(name)) {
          seen.set(name, []);
        }
        seen.get(name)!.push(index);
      }
    });

    seen.forEach((indices, name) => {
      if (indices.length > 1) {
        indices.slice(1).forEach((index) => {
          ctx.addIssue({
            code: z.ZodIssueCode.custom,
            message: "This language has already been added",
            path: ["languages", index, "language_name"],
          });
        });
      }
    });
  });
```

---

## 7. API Calls with TanStack Query

The `usePagination` hook generates pagination ranges with ellipsis support:

```typescript
// src/hooks/use-pagination.ts
export const usePagination = ({
  totalCount,
  pageSize,
  siblingCount = 1,
  currentPage,
}: {
  totalCount: number;
  pageSize: number;
  siblingCount?: number;
  currentPage: number;
}) => {
  const totalPageCount = Math.ceil(totalCount / pageSize);

  const paginationRange = useMemo(() => {
    const totalPageNumbers = siblingCount + 5;

    if (totalPageNumbers >= totalPageCount) {
      return Array.from({ length: totalPageCount }, (_, i) => i + 1);
    }

    const leftSiblingIndex = Math.max(currentPage - siblingCount, 1);
    const rightSiblingIndex = Math.min(
      currentPage + siblingCount,
      totalPageCount
    );

    const showLeftEllipsis = leftSiblingIndex > 2;
    const showRightEllipsis = rightSiblingIndex < totalPageCount - 2;

    const range: unknown[] = [];

    if (showLeftEllipsis) {
      range?.push(1, "...");
    } else {
      range.push(...Array.from({ length: leftSiblingIndex }, (_, i) => i + 1));
    }

    for (let i = leftSiblingIndex; i < rightSiblingIndex; i++) {
      range.push(i + 1);
    }

    if (showRightEllipsis) {
      range.push("...", totalPageCount);
    } else {
      range.push(
        ...Array.from(
          { length: totalPageCount - rightSiblingIndex },
          (_, i) => i + rightSiblingIndex + 1
        )
      );
    }

    return range;
  }, [totalCount, pageSize, siblingCount, currentPage]);

  return paginationRange;
};
```

**Usage:**

```typescript
// src/components/common/pagination.tsx
const paginationRange = usePagination({
  totalCount: 100,
  pageSize: 10,
  siblingCount: 1,
  currentPage: 5,
});
// Returns: [1, "...", 4, 5, 6, "...", 10]
```

---

## 9. Custom Hooks

### Pagination Hook

The `usePagination` hook generates pagination ranges with ellipsis support:

```typescript
// src/hooks/use-pagination.ts
export const usePagination = ({
  totalCount,
  pageSize,
  siblingCount = 1,
  currentPage,
}: {
  totalCount: number;
  pageSize: number;
  siblingCount?: number;
  currentPage: number;
}) => {
  const totalPageCount = Math.ceil(totalCount / pageSize);

  const paginationRange = useMemo(() => {
    const totalPageNumbers = siblingCount + 5;

    if (totalPageNumbers >= totalPageCount) {
      return Array.from({ length: totalPageCount }, (_, i) => i + 1);
    }

    const leftSiblingIndex = Math.max(currentPage - siblingCount, 1);
    const rightSiblingIndex = Math.min(
      currentPage + siblingCount,
      totalPageCount
    );

    const showLeftEllipsis = leftSiblingIndex > 2;
    const showRightEllipsis = rightSiblingIndex < totalPageCount - 2;

    const range: unknown[] = [];

    if (showLeftEllipsis) {
      range?.push(1, "...");
    } else {
      range.push(...Array.from({ length: leftSiblingIndex }, (_, i) => i + 1));
    }

    for (let i = leftSiblingIndex; i < rightSiblingIndex; i++) {
      range.push(i + 1);
    }

    if (showRightEllipsis) {
      range.push("...", totalPageCount);
    } else {
      range.push(
        ...Array.from(
          { length: totalPageCount - rightSiblingIndex },
          (_, i) => i + rightSiblingIndex + 1
        )
      );
    }

    return range;
  }, [totalCount, pageSize, siblingCount, currentPage]);

  return paginationRange;
};
```

**Usage:**

```typescript
// src/components/common/pagination.tsx
const paginationRange = usePagination({
  totalCount: 100,
  pageSize: 10,
  siblingCount: 1,
  currentPage: 5,
});
// Returns: [1, "...", 4, 5, 6, "...", 10]
```

### Other Custom Hooks

- `use-location-data.ts` - Location data management
- `use-sse-notification.ts` - Server-Sent Events for real-time notifications

---

## 8. Zustand Store Management

We use **Zustand** for global state management with a consistent pattern across all stores.

### Store Structure Pattern

All Zustand stores follow this structure:

```typescript
interface StoreState {
  // State properties
  property1: Type;
  property2: Type;

  // Actions grouped in an object
  actions: {
    action1: (param: Type) => void;
    action2: () => void;
  };
}

const useStore = create<StoreState>((set) => ({
  // Initial state
  property1: initialValue,
  property2: initialValue,

  // Actions
  actions: {
    action1: (param) => set({ property1: param }),
    action2: () => set({ property2: newValue }),
  },
}));

// Selector hooks
export const useProperty1 = () => useStore((state) => state.property1);
export const useProperty2 = () => useStore((state) => state.property2);
export const useStoreActions = () => useStore((state) => state.actions);
```

### Authentication Store

```typescript
// src/store/auth-store.ts
interface AuthState {
  user: Record<string, unknown> | null;
  isAuthenticated: boolean;
  role: "employer" | "job_seeker" | null;
  actions: {
    setAuthToken: (token: LoginResponse) => void;
    setAuthUserDetails: (userDetails: Record<string, unknown>) => void;
    logout: () => void;
  };
}

const useAuthStore = create<AuthState>((set) => {
  const token = getToken(); // Get from session storage

  let decodedToken = null;
  let role = null;
  let isAuthenticated = false;

  if (token) {
    decodedToken = jwtDecode(token);
    role = decodedToken.role;
    isAuthenticated = true;
    axiosInstance.defaults.headers.common.Authorization = `Bearer ${token}`;
  }

  return {
    user: decodedToken,
    isAuthenticated: isAuthenticated,
    role: role,
    actions: {
      setAuthToken: (token: LoginResponse) => {
        if (token?.access_token) {
          setToken(token.access_token);
          const decodedToken = jwtDecode(token.access_token);
          set({
            isAuthenticated: true,
            role: decodedToken?.role || token?.user_role,
          });
        }
      },
      setAuthUserDetails: (userDetails: Record<string, unknown>) => {
        set({ user: userDetails });
      },
      logout: () => {
        removeToken();
        delete axiosInstance.defaults.headers.common["Authorization"];
        set({ user: null, isAuthenticated: false, role: null });
      },
    },
  };
});

// Selector hooks
export const useAuthUser = () => useAuthStore((state) => state.user);
export const useAuthIsAuthenticated = () =>
  useAuthStore((state) => state.isAuthenticated);
export const useAuthRole = () => useAuthStore((state) => state.role);
export const useAuthActions = () => useAuthStore((state) => state.actions);
```

**Usage:**

```typescript
const isAuthenticated = useAuthIsAuthenticated();
const userRole = useAuthRole();
const { setAuthToken, logout } = useAuthActions();
```

### Layout Store

```typescript
// src/store/layout-store.ts
interface DashboardSidebarState {
  isOpenSidebar: boolean;
  isLogoutConfirm: boolean;
  actions: {
    setIsOpenSidebar: (isOpenSidebar: boolean) => void;
    setIsLogoutConfirm: (isLogoutConfirm: boolean) => void;
  };
}

const useDashboardSidebarStore = create<DashboardSidebarState>((set) => ({
  isOpenSidebar: false,
  isLogoutConfirm: false,
  actions: {
    setIsOpenSidebar: (isOpenSidebar) => set(() => ({ isOpenSidebar })),
    setIsLogoutConfirm: (isLogoutConfirm) => set(() => ({ isLogoutConfirm })),
  },
}));

export const useIsOpenSidebar = () =>
  useDashboardSidebarStore((state) => state.isOpenSidebar);
export const useIsLogoutConfirm = () =>
  useDashboardSidebarStore((state) => state.isLogoutConfirm);
export const useDashboardSidebarActions = () =>
  useDashboardSidebarStore((state) => state.actions);
```

### Notification Store

```typescript
// src/store/notification-store.ts
type NotificationStore = {
  notifications: Array<NotificationItemI>;
  unreadCount: number;
  actions: {
    appendNotification: (notification: NotificationItemI) => void;
    setNotifications: (notifications: Array<NotificationItemI>) => void;
    setUnreadCount: (count: number) => void;
  };
};

export const useNotificationStore = create<NotificationStore>((set) => ({
  notifications: [],
  unreadCount: 0,
  actions: {
    appendNotification: (notification: NotificationItemI) =>
      set((state) => ({
        notifications: [notification, ...state.notifications],
        unreadCount: state.unreadCount + 1,
      })),
    setNotifications: (notifications) => set({ notifications }),
    setUnreadCount: (count) => set({ unreadCount: count }),
  },
}));

export const useNotifications = () =>
  useNotificationStore((state) => state.notifications);
export const useUnreadCount = () =>
  useNotificationStore((state) => state.unreadCount);
export const useNotificationActions = () =>
  useNotificationStore((state) => state.actions);
```

### Store Best Practices

1. **Group Actions**: Always group actions in an `actions` object
2. **Selector Hooks**: Export individual selector hooks for each state property
3. **Initialization**: Handle initial state from session storage or API if needed
4. **Type Safety**: Use TypeScript interfaces for all store states
5. **Separation**: Keep stores focused on a single domain (auth, layout, notifications)

---

## Code Examples

### Complete Example: Language CRUD with Form Validation

```typescript
// 1. Schema (src/schema/registration-schema.ts)
export const languageSchema = z.object({
  language_name: z.string().min(1, "Language name is required"),
  proficiency: z.enum(
    ["beginner", "intermediate", "proficient", "advanced", "native"],
    { message: "Invalid proficiency level" }
  ),
});

// 2. API Hooks (src/apis/jobseeker.ts)
export const useCreateLanguage = () => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async (data: LanguageParams) => {
      const res = await axiosInstance.post(endPoints.jobseeker.languages, data);
      return res.data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["jobseekerMe"] });
    },
  });
};

// 3. Form Component (src/sections/profile/forms/language-form.tsx)
export default function LanguageForm({ languageObject, onClose }) {
  const { mutate: createLanguage } = useCreateLanguage();
  const { mutate: updateLanguage } = useUpdateLanguage();

  const methods = useForm<LanguageFormData>({
    resolver: zodResolver(languageSchema),
    mode: "onChange",
    defaultValues: {
      language_name: languageObject?.language_name || "",
      proficiency: languageObject?.proficiency || "native",
    },
  });

  const onSubmit = (data: LanguageFormData) => {
    if (languageObject?.id) {
      updateLanguage(
        { id: languageObject.id, ...data },
        {
          onSuccess: () => {
            toast.success("Language updated");
            onClose();
          },
        }
      );
    } else {
      createLanguage(data, {
        onSuccess: () => {
          toast.success("Language created");
          onClose();
        },
      });
    }
  };

  return (
    <FormProvider {...methods}>
      <form onSubmit={handleSubmit(onSubmit)}>
        <SelectInput name="language_name" options={languageOptions} />
        <SelectInput name="proficiency" options={proficiencyOptions} />
        <button type="submit">Save</button>
      </form>
    </FormProvider>
  );
}
```

---

## Best Practices

### API Calls

1. Always use TanStack Query hooks for API calls
2. Invalidate related queries after mutations
3. Use `enabled` option for conditional queries
4. Handle loading and error states properly
5. Use TypeScript types for all API responses

### Forms

1. Define Zod schemas for all forms
2. Use `FormProvider` to wrap forms
3. Use RHF components from `src/components/form/`
4. Validate on `onChange` for real-time feedback
5. Show clear error messages

### State Management

1. Use Zustand for global state
2. Group actions in an `actions` object
3. Export selector hooks for each state property
4. Keep stores focused on single domains
5. Initialize from session storage when needed

### Components

1. Keep components small and focused
2. Use TypeScript for all props
3. Follow consistent naming conventions
4. Extract reusable logic into custom hooks
5. Use Tailwind CSS for styling

---

## Additional Resources

- [TanStack Query Documentation](https://tanstack.com/query/latest)
- [React Hook Form Documentation](https://react-hook-form.com/)
- [Zod Documentation](https://zod.dev/)
- [Zustand Documentation](https://zustand-demo.pmnd.rs/)
- [Next.js Documentation](https://nextjs.org/docs)

---

_Last Updated: 2024_
